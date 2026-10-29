# 车来了 MCP

一个基于 MCP 的车来了实时公交服务器，让大语言模型可以查询国内的公交、地铁实时数据 —— 包括线路时刻表、车辆实时位置、附近站点、关键词搜索、以及公交+地铁的换乘路线规划。

回答 "我的公交还有多久到站" 这类问题。**无需登录、无需任何账号配置**，开箱即用。

> Fork of [PeanutSplash/chelaile-mcp](https://github.com/PeanutSplash/chelaile-mcp)，修复了
> `bus_get_timetable` 在服务日尾部查不到班次的问题，见[下文](#发车预测说明本-fork-的改动)。

## 安装

本 fork 未发布到 npm，直接从 GitHub 构建（仓库为私有，需先完成 GitHub 认证）：

```bash
git clone https://github.com/KamijoToma/chelaile-mcp.git
cd chelaile-mcp
bun install
npm run build
```

然后在 MCP 客户端里指向 `dist/index.js` 的绝对路径：

```json
{
  "mcpServers": {
    "chelaile": {
      "command": "node",
      "args": ["/absolute/path/to/chelaile-mcp/dist/index.js"]
    }
  }
}
```

Claude Code 等价写法：

```bash
claude mcp add chelaile -- node /absolute/path/to/chelaile-mcp/dist/index.js
```

也可以先 `npm link`，之后直接 `chelaile-mcp` 启动。

重启客户端后，工具会以 `chelaile` 为前缀出现。

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
2. 当天预测只覆盖**未来约 1.5–2 小时的滚动窗口**，服务日尾部（如 21:00 末班）永远不在其中。

实测同一线路不同时刻的窗口末端：

| 查询时刻 | 当天预测班数 | 当天预测末班 | 官方末班 |
| --- | --- | --- | --- |
| 16:02 | 31 | 17:35 | 21:00 |
| 18:10 | 35 | 20:00 | 21:00 |

窗口随当前时间前移，所以「19:10 之后下一班」在下午查不到、在傍晚查得到 —— 但**回答必须一致**，不能取决于查询时刻。

本 fork 的处理：

- **不再传 `tabType`** —— 上游因此额外返回 `yesterdayTimetable` /
  `beforeYesterdayTimetable`，即前两个服务日的**完整实际班次**（到 21:00）。
- 解析上游嵌套结构 `timetable.timetable[]`（原版用 `Array.isArray` 判断，恒为 `null`）。
- 新增 `at_time` 参数，按「指定时刻之后的第一班」计算 `nextDeparture`；
  当天预测够不到时自动回落到昨日实际班次，`nextDepartureSource` 标明数据来源。

### 示例

```jsonc
// bus_get_timetable { city_id: "<cityId>", line_id: "<lineId>", line_no: "<lineNo>",
//                     direction: "1", at_time: "19:10" }
{
  "nextDeparture": "20:00",
  "nextDepartureSource": "yesterday-actual",   // 或 today-prediction，见下表
  "lastPredictedDeparture": "20:00",
  "yesterdayDepartures": ["06:30", "...", "19:10", "20:00", "21:00"],
  "segments": [{ "fromTime": "06:30", "toTime": "10:00", "intervalMin": 18 }]
}
```

「19:10 之后第一班 = 20:00」这个结论稳定，只有来源标签随查询时刻变化
（当天预测是滚动窗口，覆盖到哪算到哪）：

| 查询时刻 | 当天预测末班 | `nextDeparture` | `nextDepartureSource` |
| --- | --- | --- | --- |
| 16:02 | 17:35 | 20:00 | `yesterday-actual` |
| 18:10 | 20:00 | 20:00 | `today-prediction` |

`at_time` 超出所有数据覆盖范围时返回 `null` 而非猜一个值：

| `at_time` | `nextDeparture` |
| --- | --- |
| `05:00` | `06:30`（今日首班） |
| `20:30` | `21:00`（昨日末班） |
| `21:30` | `null`（已过末班） |

不传 `at_time` 时按当前北京时间取下一班。

## License

MIT
