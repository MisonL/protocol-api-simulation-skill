# 仓库指南

## 项目结构与模块组织

本仓库是一个轻量级 skill 包，核心文件是 `protocol-api-simulation/SKILL.md`。该文件定义 skill 元数据、适用范围、安全边界、执行流程和输出要求。参考资料位于 `protocol-api-simulation/references/`，当前包含两类内容：`protocol-api-simulation/references/codex-console-lessons.md` 用于沉淀具体产品经验，`protocol-api-simulation/references/security-lab-lessons.md` 用于沉淀通用实验环境、观测与重放模式。仓库目前没有 `src/`、`tests/` 或资源目录，除非明确扩展结构，否则变更应聚焦 Markdown 文档。

## 构建、验证与开发命令

本仓库没有传统构建流程，日常维护以文档检查为主：

- `sed -n '1,120p' protocol-api-simulation/SKILL.md`：快速查看主 skill 定义。
- `sed -n '1,120p' protocol-api-simulation/references/security-lab-lessons.md`：单独复查通用实验参考。
- `wc -w AGENTS.md protocol-api-simulation/SKILL.md`：控制文档规模，避免无谓膨胀。
- `rg -n '^## ' protocol-api-simulation/SKILL.md AGENTS.md`：检查标题结构是否清晰。
- `rg -n '/Users/|/Volumes/' README.md protocol-api-simulation/SKILL.md protocol-api-simulation/references/*.md AGENTS.md`：提交前排查本机路径泄漏。
- `git diff --check`：检查空白符和补丁格式问题。
- `cd protocol-api-simulation && uv run --with pyyaml python <path-to-skill-creator>/scripts/quick_validate.py .`：校验 skill frontmatter 与命名规则。

如果后续新增脚本或工具链，必须把准确命令补充到本节，不要依赖隐含约定。

## 编写风格与命名约定

文档统一使用中文，表达要直接、简洁、可执行。使用标准 Markdown 标题和单层列表，避免装饰性 Unicode 符号和冗长解释。顶层贡献者文档保持大写命名，例如 `AGENTS.md`；skill 主文件保留 `protocol-api-simulation/SKILL.md`。新增文件时优先使用描述性名称，例如 `docs/redirect-trace-notes.md` 或 `protocol-api-simulation/references/oauth-state-cases.md`。

## 验证要求

本仓库的验证以文档一致性为主。提交前至少确认：

- 标题、列表和表格渲染正常。
- `README.md`、`AGENTS.md` 与 `protocol-api-simulation/SKILL.md` 对仓库现状的描述一致。
- 示例和约束仍然围绕“已授权协议测试、会话一致性、PKCE、跳转追踪、风控边界”。
- `protocol-api-simulation/references/` 下的文件名、链接和用途说明在结构调整后仍然正确。
- 对 `protocol-api-simulation/SKILL.md` 做了实质修改时，重新运行 `quick_validate.py`。

如果未来引入脚本或可执行示例，再新增对应的 `tests/` 目录，并在本文件中写清如何运行。

## 提交与合并请求规范

当前提交历史采用短小、单一职责的说明，例如 `docs: add protocol-api-simulation skill`、`docs: add security lab reference patterns`、`docs: refresh repository documentation`。提交应只覆盖一类文档变化，并使用简短的祈使句摘要。合并请求需要说明本次修改了什么指导内容、运行了哪些校验命令，以及影响范围是主 `protocol-api-simulation/SKILL.md`、仓库说明文档，还是仅参考资料。

