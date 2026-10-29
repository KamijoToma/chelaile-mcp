# Security Policy

## Supported Versions

本项目处于活跃维护中，跟随 `main` 分支的最新版本。

| Version | Supported          |
| ------- | ------------------ |
| 1.2.x   | :white_check_mark: |
| < 1.2   | :x:                |

## Reporting a Vulnerability

请**不要**通过公开 Issue 报告安全漏洞。请使用 GitHub 的
[Private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability)
功能提交私密报告。

如果该功能不可用，请通过仓库 Issues 页面联系维护者并说明「security report」，
维护者会转交私密渠道。

### 报告内容建议

- 受影响版本 / commit
- 复现步骤
- 影响范围与严重程度
- 任何已知的缓解措施

维护者会在收到报告后尽快确认，通常在 7 天内给出初步回应。

## 威胁模型

本项目是一个**本地 stdio MCP server**，由用户在自己的机器上运行，
通过 stdio 与 MCP 客户端通信。本身不监听网络端口、不存储用户数据、
不持久化任何会话信息。

需要注意的是：

- **上游接口凭据内置在代码中**。`src/constants.ts` 里的签名盐值、AES 密钥与
  微信小程序 appid 来自上游仓库（[PeanutSplash/chelaile-mcp](https://github.com/PeanutSplash/chelaile-mcp)），
  并**已随该公开仓库对外可见**。它们不是本项目生成的私密凭据，也**无法通过本项目撤销**。
  任何人都可以据此构造对上游的合法请求，因此**请勿依赖它们作为访问控制手段**。
- **上游接口非公开 API**，可能随时变更或封禁。本项目不做可用性或准确性的安全承诺。
- **第三方依赖**。运行时依赖仅 `@modelcontextprotocol/sdk` 与 `zod`；开发依赖包含
  `release-it`、`typescript`、`tsx` 等。定期执行 `bun audit` / `npm audit` 建议纳入 CI。
- **运行时网络出口**：仅访问上游车来了域名与 `ip-api.com`（`bus_get_my_location` 使用）。

## 不在安全范围内的内容

- 上游服务的可用性、正确性或数据准确性
- 因高频调用上游导致的封禁或限流
- 用户自身的 MCP 客户端配置与权限管理
