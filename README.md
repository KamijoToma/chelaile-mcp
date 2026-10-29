# 车来了 MCP

一个基于 MCP 的车来了实时公交服务器，让大语言模型可以查询国内的公交、地铁实时数据 —— 包括线路时刻表、车辆实时位置、附近站点、关键词搜索、以及公交+地铁的换乘路线规划。

回答 "我的公交还有多久到站" 这类问题。**无需登录、无需任何账号配置**，开箱即用。

[![npm version](https://img.shields.io/npm/v/chelaile-mcp-server?color=cb3837&logo=npm)](https://www.npmjs.com/package/chelaile-mcp-server)
[![Install in Cursor](https://img.shields.io/badge/Cursor-一键安装-000000?style=flat-square&logo=cursor&logoColor=white)](cursor://anysphere.cursor-deeplink/mcp/install?name=chelaile&config=eyJjb21tYW5kIjoibnB4IiwiYXJncyI6WyIteSIsImNoZWxhaWxlLW1jcC1zZXJ2ZXIiXX0=)
[![Install in VS Code](https://img.shields.io/badge/VS_Code-一键安装-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode:mcp/install?%7B%22name%22%3A%22chelaile%22%2C%22command%22%3A%22npx%22%2C%22args%22%3A%5B%22-y%22%2C%22chelaile-mcp-server%22%5D%7D)
[![Install in VS Code Insiders](https://img.shields.io/badge/VS_Code_Insiders-一键安装-24bfa5?style=flat-square&logo=visualstudiocode&logoColor=white)](vscode-insiders:mcp/install?%7B%22name%22%3A%22chelaile%22%2C%22command%22%3A%22npx%22%2C%22args%22%3A%5B%22-y%22%2C%22chelaile-mcp-server%22%5D%7D)

## 安装

### 一键安装

点击上面的徽章即可在 Cursor / VS Code / VS Code Insiders 中一键安装。

### 在 Claude Code 中使用

```bash
claude mcp add chelaile -- npx -y chelaile-mcp-server
```

或者手动编辑 `~/.claude/mcp_servers.json`（或项目级的 `.mcp.json`）：

```json
{
  "mcpServers": {
    "chelaile": {
      "command": "npx",
      "args": ["-y", "chelaile-mcp-server"]
    }
  }
}
```

### 在 Claude Desktop 中使用

编辑 `~/Library/Application Support/Claude/claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "chelaile": {
      "command": "npx",
      "args": ["-y", "chelaile-mcp-server"]
    }
  }
}
```

重启对应客户端后，工具会以 `chelaile` 为前缀出现。

## 工具一览

| 工具                    | 用途                                    |
| ----------------------- | --------------------------------------- |
| `bus_list_cities`       | 列出支持的城市及其 ID                   |
| `bus_get_city_config`   | 某城市的刷新间隔限制与展示策略          |
| `bus_get_my_location`   | 基于调用方 IP 估算位置（精度到城市级）  |
| `bus_reverse_geocode`   | WGS-84 经纬度 → 中文地址                |
| `bus_search`            | 按关键词混合搜索：线路 + 站点 + POI     |
| `bus_search_more`       | 按某一分类分页"查看更多"                |
| `bus_get_nearby_stops`  | 某 GPS 坐标附近的站点及到站预计时间     |
| `bus_get_stop_detail`   | 某站点经过的所有线路 + 附近的地铁       |
| `bus_get_line_detail`   | 某线路的完整站点列表 + 当前车辆         |
| `bus_get_line_route`    | 某线路的地图轨迹坐标                    |
| `bus_get_line_realtime` | 某线路即将到达某站点的车辆实时信息      |
| `bus_list_line_buses`   | 某线路上所有车辆的位置与载客率          |
| `bus_get_timetable`     | 首站发车预测（含末班车，支持 `at_time` 查指定时刻后的下一班） |
| `bus_refresh_lines`     | 一次性批量刷新多个 (线路, 站点) 对      |
| `bus_plan_transit`      | 两个 GCJ-02 坐标之间的公交+地铁换乘规划 |

每个工具都支持 `response_format: "markdown" | "json"`（默认 `markdown`），返回 JSON 时同时附带 `structuredContent`。

## 发车预测说明（本 fork 的改动）

`bus_get_timetable` 回答「始发站下一班几点发车」。

上游 `bus/line!preStartTimetableNew.action` 有两个坑，原版都没处理：

1. 传入 `tabType=0` 会让上游**只返回当天预测**，屏蔽另外两份完整数据。
2. 当天预测只覆盖服务日的前 ~80%（实测末班 21:00 的线路，预测只到 17:35）。

本 fork 的处理：

- **不再传 `tabType`** —— 上游因此额外返回 `yesterdayTimetable` /
  `beforeYesterdayTimetable`，即前两个服务日的**完整实际班次**（到 21:00）。
- 解析上游嵌套结构 `timetable.timetable[]`（原版用 `Array.isArray` 判断，恒为 `null`）。
- 新增 `at_time` 参数，按「指定时刻之后的第一班」计算 `nextDeparture`；
  当天预测不够时自动回落到昨日实际班次，并用 `nextDepartureSource` 标明来源。

### 示例

```jsonc
// bus_get_timetable { city_id: "<cityId>", line_id: "<lineId>", line_no: "<lineNo>",
//                     direction: "1", at_time: "19:10" }
{
  "nextDeparture": "20:00",
  "nextDepartureSource": "yesterday-actual",
  "lastPredictedDeparture": "17:35",
  "firstStopDepartures": ["06:30", "06:45", "...", "17:15", "17:35"],
  "yesterdayDepartures": ["06:30", "...", "19:10", "20:00", "21:00"],
  "segments": [{ "fromTime": "06:30", "toTime": "10:00", "intervalMin": 18 }]
}
```

即：该方向 19:10 之后的第一班发车是 **20:00**。

不传 `at_time` 时按当前北京时间取下一班。

## License

MIT
