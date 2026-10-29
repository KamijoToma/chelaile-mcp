# Contributing

感谢你愿意为这个项目出力。

本仓库是 [PeanutSplash/chelaile-mcp](https://github.com/PeanutSplash/chelaile-mcp) 的 fork。
上游的通用开发约定同样适用；fork 特有的部分见下文。

## 开发环境

- **Node.js** >= 18
- **Bun** >= 1.2（用于运行测试与 lockfile）

```bash
bun install
bun run build
```

## 提交前检查

```bash
bun test tests/unit    # 单元测试，必须全绿
npx tsc --noEmit       # 类型检查，必须无错误
```

端到端测试会真实请求上游接口，**只在改动涉及请求参数或响应解析时才需要跑**：

```bash
CHELAILE_E2E=1 bun test tests/e2e
```

## 提交规范

项目使用 [Conventional Commits](https://www.conventionalcommits.org/)，
changelog 由 `release-it` + `conventional-changelog` 自动生成：

```
feat(timetable): add at_time argument
fix(timetable): fall back to yesterday's schedule when predictions end
docs(readme): clarify coordinate systems
refactor(http-client): extract envelope parsing
```

类型：`feat` / `fix` / `docs` / `refactor` / `perf` / `test` / `build` / `ci` / `chore`。
影响工具行为的改动请用对应工具名作 scope，例如 `fix(timetable): ...`。

## 约定

### 上游是唯一事实来源

上游车来了接口**不是公开 API**，随时可能变更。响应解析应只依赖实际观察到的字段，
不要基于猜测添加字段。每个工具的响应结构都有对应的 zod schema 与 e2e 契约测试 ——
**上游 schema 漂移时，是 e2e 测试先失败**，这是预期行为。

新增或修改响应字段时，请同步更新：

1. `src/types.ts` 中的类型定义
2. `src/tools/<tool>.ts` 中的 `reshape*` 函数与精简后的 lean 类型
3. `tests/unit/tools/<tool>.test.ts` 中的单元测试
4. `tests/e2e/api.e2e.test.ts` 中对应工具的 zod schema（若字段有变化）

### 坐标系统

上游同时存在 WGS-84 与 GCJ-02 两套坐标，工具之间混用会导致百米级偏差。
新增工具时必须在参数描述里明确标注坐标系统，并沿用现有约定：

- `bus_reverse_geocode`、`bus_get_nearby_stops` 接收 **WGS-84**
- `bus_plan_transit` 接收 **GCJ-02**
- `bus_search` 返回的 `pois[].lat/lng` 是 **GCJ-02**，`stations[].lat/lng` 是 **WGS-84**

### 输出体积

工具输出通过 `format.ts` 的 `truncate()` 统一截断到 25000 字符。
Markdown 模式**刻意不返回** `structuredContent` —— 部分客户端（如 Claude Code）
会优先渲染 `structuredContent`，从而绕过人类可读文本。JSON 模式才附带。

### 不要提交的内容

- 真实用户的坐标、IP、地址等个人信息
- 具体线路编号、城市编号、内部 `lineId` 等可定位到某人常用线路的信息
- 任何真实的 token、密钥或 session id

## 提交 Pull Request

1. 从 `main` 切出分支：`git checkout -b fix/short-description`
2. 完成改动并补上测试
3. 跑通上面的提交前检查
4. 推送并创建 PR，描述中说明**问题**与**解决方式**，而非只说改了哪些文件

## 联系

- 上游问题与上游功能建议 → [PeanutSplash/chelaile-mcp](https://github.com/PeanutSplash/chelaile-mcp)
- 本 fork 特有的问题 → 本仓库 Issues
