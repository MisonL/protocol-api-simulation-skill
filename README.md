# protocol-api-simulation-skill

这是一个面向协议链路复现与审计的通用 Codex skill 仓库。它用于在**明确授权**前提下，复现浏览器式 API 流程，定位 OAuth、PKCE、邮件 OTP、302 跳转、cookie 演进、callback 解析、JWT/JWE 提取和强 session 一致性问题。

## 仓库结构

- `SKILL.md`
  - 主 skill 定义。包含触发条件、闭环流程、状态机、验证矩阵、观测模板和边界约束。
- `references/codex-console-lessons.md`
  - 具体项目经验参考。只在命中特定实现细节时按需加载。
- `references/security-lab-lessons.md`
  - 从 ZAP、mitmproxy、Juice Shop、CTFd 提炼的通用实验与观测模式，只保留可安全复用的技巧。
- `AGENTS.md`
  - 本仓库的执行约束与协作要求。

当前仓库是文档型 skill 仓库，没有 `src/`、`tests/` 或独立脚本目录；主要内容集中在主 skill 和按需加载的参考资料。

## 这个 skill 解决什么问题

适用场景：

- 已获授权的 OAuth 或注册/登录续跑排障
- 邮件 OTP / 外部验证链路复现
- 302 redirect chain 追踪
- callback `query` / `fragment` 解析
- challenge 材料、cookie、session token 是否串会话的审计
- 把脆弱的多步骤网页流程整理成可恢复状态机

不适用场景：

- 绕过验证码、设备绑定、限频或平台风控
- 未授权令牌抓取、批量注册、账户接管或规避检测自动化

## 使用方式

在支持 skills 的 Codex 环境中引用本目录即可。主入口是 `SKILL.md`。

最短阅读顺序：

- 首次上手先看 `SKILL.md` 里的“首轮 5 问”“启动模板”“执行模式分层”。
- 命中特定实现细节时再读 `references/codex-console-lessons.md`。
- 涉及录制、重放、隔离环境或证据工件管理时再读 `references/security-lab-lessons.md`。

典型请求示例：

- “使用这个 skill 排查为什么资源创建成功但 token 没拿到。”
- “使用这个 skill 审计这条 OAuth 302 跳转链是否保持强 session 一致性。”
- “使用这个 skill 复现邮箱 OTP 链路，并判断是否发生重复发码污染。”

## 当前内容重点

- 主 skill 已包含首轮起手问题、16 步闭环、最小证据包、执行模式分层和输出骨架。
- 参考资料当前分为两类：具体产品经验（`codex-console-lessons.md`）与通用安全实验/观测模式（`security-lab-lessons.md`）。
- 仓库当前以 Markdown 维护为主；如后续新增脚本或示例，应同步更新 `AGENTS.md` 与本 README。

## 验证

可用 `skill-creator` 自带校验脚本做结构检查：

```bash
uv run --with pyyaml python <path-to-skill-creator>/scripts/quick_validate.py .
```

当前仓库已经通过该校验。

建议在文档更新后额外做两类轻量检查：

- `git diff --check`
- `rg -n '/Users/|/Volumes/' README.md SKILL.md references/*.md AGENTS.md`

## 安全边界

本 skill 默认遵循三条硬规则：

- 先确认授权范围，再复现链路
- 风控命中即停，保留证据，不写规避逻辑
- 不伪造身份、不伪造 JWT、不重放敏感令牌
