# 车来了 MCP

一个基于 MCP 的车来了实时公交服务器，让大语言模型可以查询国内的公交、地铁实时数据 —— 包括线路时刻表、车辆实时位置、附近站点、关键词搜索、以及公交+地铁的换乘路线规划。

回答 "我的公交还有多久到站" 这类问题。**无需登录、无需任何账号配置**，开箱即用。

> **Fork 说明**
> 本仓库 fork 自 [PeanutSplash/chelaile-mcp](https://github.com/PeanutSplash/chelaile-mcp)（MIT），
> 修复了 `bus_get_timetable` 在服务日尾部查不到班次的问题，并新增 `at_time` 参数。
> 完整改动见 [CHANGELOG_FORK.md](CHANGELOG_FORK.md)，差异概览见[下文](#发车预测说明本-fork-的改动)。
> 上游版本仍发布在 npm 上（`chelaile-mcp-server`），本 fork 未发布到 npm。

## 目录

- [特性](#特性)
- [安装](#安装)
- [工具一览](#工具一览)
- [发车预测说明（本 fork 的改动）](#发车预测说明本-fork-的改动)
- [配置](#配置)
- [开发](#开发)
- [致谢](#致谢)
- [免责声明](#免责声明)
- [License](#license)

## 特性

- **15 个工具**覆盖城市、线路、站点、实时车辆、时刻表、换乘规划全链路
- **两种输出格式**：每个工具都支持 `response_format: "markdown" | "json"`，JSON 模式同时附带 `structuredContent` 供程序化消费
- **坐标系统明确**：WGS-84 与 GCJ-02 在工具描述中逐个标注，避免坐标混用
- **发车预测含末班车**：回落到前两个服务日的实际班次，回答不再随查询时刻漂移
- **无需账号**：直接复用车来了的公开接口，无 API Key、无登录态
- **纯 TypeScript**：零运行时依赖（除 MCP SDK 与 zod），`tsc` 编译即可

## 安装

本 fork 未发布到 npm，直接从 GitHub 构建：

```bash
git clone https://github.com/misakacloud/chelaile-mcp.git
cd chelaile-mcp
bun install
npm run build
```

> 仓库当前为私有仓库，克隆前需先完成 GitHub 认证（SSH key 或 PAT）。

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

### Docker

```bash
docker build -t chelaile-mcp .
docker run -i --rm chelaile-mcp
```

镜像以 stdio 方式运行，直接挂到 MCP 客户端上即可。

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

### 典型用法

```
用户：71 路还有多久到站？
  → bus_search { city_id, keyword: "71路" }        # 拿到 lineId 与方向
  → bus_get_nearby_stops { lat, lng }               # 找到附近该线路的站点
  → bus_get_line_realtime { line_id, station_id }   # 该站点的实时车辆 ETA
```

```
用户：明天早上 7 点后下一班从始发站几点发？
  → bus_get_timetable { city_id, line_id, line_no, direction, at_time: "07:00" }
```

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

## 配置

无需任何环境变量或配置文件。所有工具的参数都通过 MCP 调用传入。

唯一的外部依赖是上游接口的车来了签名参数，它们固化在 `src/constants.ts` 中。
上游接口并非公开 API，**可能随时变更或失效**；出现大面积 `status:400` 通常意味着上游调整了签名算法，需要同步上游仓库的最新实现。

## 开发

```bash
bun install
npm run build        # tsc 编译到 dist/
bun test tests/unit  # 单元测试（171 个，不联网）
npm run dev          # tsx watch 调试
```

端到端测试会真实请求上游接口，默认跳过：

```bash
CHELAILE_E2E=1 bun test tests/e2e   # 15 个用例，检测上游 schema 漂移
```

用 MCP Inspector 交互式调试：

```bash
npm run inspect
```

### 项目结构

```
src/
  index.ts          # MCP server 入口，注册全部工具
  constants.ts      # 上游签名、请求头、超时
  http-client.ts    # 签名、加密信封解密、gzip/br 解压、请求封装
  format.ts         # 输出格式化、截断、坐标取整、距离清洗
  types.ts          # 上游响应与精简响应的类型定义
  tools/
    city.ts         # 城市列表与城市配置
    geo.ts          # IP 定位、逆地理编码
    search.ts       # 关键词搜索、分类分页
    stops.ts        # 附近站点、站点详情
    lines.ts        # 线路详情、实时车辆、轨迹、时刻表、批量刷新
    transit.ts      # 公交+地铁换乘规划
tests/
  unit/             # 171 个单元测试，纯离线
  e2e/              # 15 个真实上游契约测试，需 CHELAILE_E2E=1
```

### 提交规范

项目使用 [Conventional Commits](https://www.conventionalcommits.org/)，由 `release-it` + `conventional-changelog` 生成 changelog：

```
feat(timetable): add at_time argument
fix(timetable): fall back to yesterday's schedule
docs(readme): clarify coordinate systems
```

## 致谢

- 上游作者 [PeanutSplash](https://github.com/PeanutSplash/chelaile-mcp) —— 原始实现与全部工具，本 fork 基于其 `2ba26c3`
- [Model Context Protocol](https://modelcontextprotocol.io) —— MCP 协议与 TypeScript SDK
- [车来了](https://www.chelaile.net.cn) —— 数据来源

## 免责声明

- 本项目为**非官方**的第三方工具，与车来了官方无任何隶属关系。
- 数据全部来自车来了的公开接口，其准确性、时效性、可用性由上游负责，本项目不作任何保证。
- 本项目仅作技术研究与个人学习用途。**请遵守上游服务的使用条款，不要用于高频调用或商业用途。**
- 换乘规划结果仅供参考，出行请以现场实际运营为准。
- 使用者需自行确保其使用行为符合当地法律法规。

## License

[MIT](LICENSE) —— 版权归原作者所有，详见 [LICENSE](LICENSE)。
