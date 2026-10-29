# Changelog

## 1.2.0

### Fork notes

Forked from [PeanutSplash/chelaile-mcp](https://github.com/PeanutSplash/chelaile-mcp)
at `2ba26c3`. See [CHANGELOG_FORK.md](CHANGELOG_FORK.md) for the full write-up.

### Bug Fixes

* **timetable:** stop sending `tabType=0`, which suppressed the upstream's two
  full-day histories (`yesterdayTimetable` / `beforeYesterdayTimetable`); the
  rolling ~2h prediction window never contains the last trip of the day
* **timetable:** parse the nested `timetable.timetable[]` block; the previous
  `Array.isArray` check was always false, so the field was always `null`
* **timetable:** resolve `nextDeparture` against yesterday's actual schedule
  when today's predictions do not reach the requested hour, and report the
  source in `nextDepartureSource`

### Features

* **timetable:** new `at_time` argument ("HH:MM", Asia/Shanghai) to ask for the
  next departure strictly after a given time instead of the current clock
* **timetable:** expose `firstStopDepartures`, `segments` (per-window headway
  and trips), `yesterdayDepartures`, `dayBeforeDepartures`,
  `lastPredictedDeparture`, `depTitle`, `depDesc`
* **timetable:** rewritten markdown output showing the next departure, the
  per-window headways, and yesterday's full-day actual departures

## 1.1.0 (2026-05-18)

### Features

* initial chelaile-mcp-server ([24e0444](https://github.com/PeanutSplash/chelaile-mcp/commit/24e0444b6bbb48afcae1a7eef68e5fb248398faa))
