# Fork 变更说明

基于 [PeanutSplash/chelaile-mcp](https://github.com/PeanutSplash/chelaile-mcp) @ 2ba26c3（v1.1.0）。

## 修复：`bus_get_timetable` 发车预测不完整

### 问题

原版对一条首班 06:30 / 末班 21:00 的线路在下午查询时，
19:10 之后最早的一班（20:00）完全查不到 —— 当天预测的滚动窗口够不到。

两个独立缺陷叠加：

**缺陷 1 — 屏蔽了完整数据。**
请求固定携带 `tabType=0`。实测该参数使上游 `bus/line!preStartTimetableNew.action`
仅返回 `{line, timeTableType, timetable}`。

不传 `tabType` 时，同一接口返回 5 个 key：

| key | 内容 | dir=1 实测 |
| --- | --- | --- |
| `timetable` | 当天预测（滚动 ~2h） | 31 班，末班 17:35（16:02 实测）|
| `yesterdayTimetable` | 昨日实际 | 36 班，末班 **21:00** |
| `beforeYesterdayTimetable` | 前日实际 | 36 班，末班 **20:59** |

当天预测是**滚动窗口**（约未来 1.5–2 小时），16:02 查时末班 17:35，18:10 查时末班 20:00；
官方末班 21:00 始终不在窗口内。所以不传 `tabType` 拿到的前两日完整班次是必需的。

**缺陷 2 — 解析形状错误。**
`timetable` 是对象 `{depTitle, nearDepTime, timetable: [...]}`，
原版 `Array.isArray(raw.timetable)` 判否，恒返回 `timetable: null`。

### 修复

`src/tools/lines.ts`：

- 去掉请求里的 `tabType: "0"`。
- 新增 `asTimetableBlock` / `blockSegments` / `segmentTimes` / `blockDepartures`
  解析嵌套块（含 `schedules[].time` 与 `times[]` 两种上游写法）。
- `reshapeTimetable(raw, atTime?)` 输出：
  - `firstStopDepartures` — 当天预测全部班次
  - `segments[]` — 分时段（起止 + 间隔 + 班次）
  - `yesterdayDepartures` / `dayBeforeDepartures` — 前两日完整实际班次
  - `nextDeparture` + `nextDepartureSource`（`today-prediction` | `yesterday-actual`）
  - `lastPredictedDeparture` — 当天预测的末班，便于判断是否需要回落
- 当天预测覆盖不到查询时刻时，自动回落到昨日/前日实际班次。
- 新增 `at_time`（`HH:MM`，北京时间）参数，语义是**严格晚于**该时刻的第一班；
  不传时按当前北京时间取「不早于现在」的第一班。
- `renderTimetable` 重写：输出下一班、时段间隔、昨日完整班次。

### 验证

```
$ bus_get_timetable { city_id:"<cityId>", line_id:"<lineId>",
                      line_no:"<lineNo>", direction:"1", at_time:"19:10" }
{
  "nextDeparture": "20:00",
  "nextDepartureSource": "today-prediction" | "yesterday-actual",
  "lastPredictedDeparture": "20:00",
  "yesterdayDepartures": ["06:30", ..., "19:10", "20:00", "21:00"]
}
```

`nextDeparture` 恒为 20:00，`nextDepartureSource` 依当天预测窗口覆盖范围变化。

边界：

| `at_time` | `nextDeparture` | 来源 |
| --- | --- | --- |
| 05:00 | 06:30 | today-prediction |
| 19:10 | 20:00 | today-prediction 或 yesterday-actual |
| 20:30 | 21:00 | yesterday-actual |
| 21:30 | `null` | 已过末班 |

### 测试

- `bun test tests/unit` → 171 pass
- `CHELAILE_E2E=1 bun test tests/e2e` → 15 pass（真实上游）
- `tsc --noEmit` 无错误

替换了 4 个围绕旧行为（`timetable: null` / "3 schedule entries"）的单元测试，
新增 `at_time` 校验与历史回落断言。
