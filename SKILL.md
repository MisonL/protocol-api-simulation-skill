---
name: protocol-api-simulation
description: For authorized browser-protocol interoperability testing and stateful flow reproduction with curl_cffi. Use when reproducing OAuth or signup/login continuations, challenge-token preflights, PKCE, email or out-of-band verification, 302 redirect chains, cookie or session extraction, callback parsing, JWT or JWE artifact analysis, or when converting a fragile multi-step web flow into a resumable state machine without bypassing risk controls.
---

# Protocol API Simulation

## 何时使用

在以下场景触发本技能：

- 已获得书面授权，需要复现实站或沙箱中的浏览器协议链路。
- 需要用 `curl_cffi` 模拟浏览器级传输特征，定位 OAuth、注册、验证邮件、302 跳转、JWT 提取等问题。
- 需要把易碎的多步骤流程整理成可重试、可恢复、可审计的状态机。

以下场景不适用：

- 绕过验证码、设备绑定、限频、账号风控或平台反滥用控制。
- 未经授权抓取令牌、批量注册、账户接管、凭证撞库或任何规避检测的自动化。

## 硬边界

- 先确认授权范围：目标域名、租户、账号类型、请求速率、可变更资源、日志留存要求。
- 风控命中后优先保留证据和停止变更，不写“自动继续闯关”的逻辑。
- 不伪造身份、不伪造 JWT、不重放敏感令牌、不使用代理池或打码服务规避控制。
- 如果用户明确要求“绕过风控”，拒绝该部分，并改为输出合规测试或防护评估方案。

## 核心立场

- 引擎基线：使用 `curl_cffi`，选择单一且稳定的浏览器 profile，例如 `impersonate="chrome"` 或目标环境对应的固定版本，目的是复现已授权浏览器客户端的传输层行为，而不是隐藏自动化身份。
- Session 强一致性：从第一跳到令牌交换，全程复用同一会话上下文，包括 cookie jar、连接池、PKCE 材料、挑战工件、重定向历史和邮件关联键。
- 成功优先判定：一旦 Step 12 的物理创建动作已经被服务端确认成功，立即把该动作视为已提交，后续只做收尾、取证和幂等保护，不再回滚猜测。
- 多段 JWT 自动探测解析：自动从响应头、正文、URL、片段、Cookie、302 Location 中提取候选令牌，解析声明但不信任未验证内容。

## 三分模型

把这类任务拆成三个层面，不要混在一起推理：

- 协议逻辑
  - 关注授权流程、公理、不变量和状态迁移。
  - 例子：`state`/`code_verifier` 是否匹配，callback 是否属于当前轮，创建成功后是否还需要补 token。
- 传输实现
  - 关注具体执行器如何发送请求和保持上下文。
  - 例子：`curl_cffi` session、浏览器 profile、HTTP 版本、cookie jar、代理、重定向处理。
- 观测器
  - 关注你依据什么判断事实成立。
  - 例子：原始请求三元组、302 链、日志、抓包、真实浏览器基线、mitmproxy 记录。

执行规则：

- 协议逻辑是控制器，传输实现是执行器，观测器是传感器。
- 如果问题出在协议逻辑，不要只调浏览器指纹或 HTTP 参数。
- 如果问题出在传输实现，不要误以为协议状态机本身错了。
- 如果事实不清，优先补观测器，再决定是否调整协议逻辑或传输实现。

## 参考导航

- 只需要高层流程、边界和输出要求时，读本文件即可。
- 如果任务命中特定产品或站点实现细节，再按需读取对应参考文件，例如 `references/codex-console-lessons.md`。
- 只有在任务真的命中这些坑型时才加载参考文件；不要把任何站点的私有行为默认套到别的目标上。

## 16 步闭环

推荐按下面的闭环组织流程，并为每一步定义输入、输出、失败分类和可重试性：

1. IP 校验：确认出口地址、地域、授权环境和目标 allowlist 状态。
2. 环境预热：访问首页或引导接口，建立首批 cookie、CSRF 和缓存上下文。
3. 基线采样：记录首轮响应头、协议版本、Set-Cookie、关键脚本入口和初始跳转链。
4. 挑战工件提取：从 HTML、内联脚本、JSON boot payload 或异步接口中提取防滥用、预检或挑战参数。
5. PKCE 挑战生成：创建 `code_verifier` 与 `code_challenge`，并把原始 verifier 持久化到会话状态。
6. 预检提交：发送预检查或初始化请求，确认会话已进入可继续状态。
7. 身份引导：提交最小必要字段，触发邮箱或外部验证链路。
8. 邮件异步轮询启动：记录关联键，进入可中断轮询。
9. 邮件轮询解析：提取验证链接、一次性代码或中间态令牌。
10. 验证链接标准化：展开包装链接，修正 HTML 编码、追踪参数和相对路径。
11. 授权链继续：带着原始 session、PKCE、挑战工件和邮件产物继续推进。
12. 物理创建：一旦服务端确认核心资源已创建，立即标记 `committed=true`。
13. JWT Payload 提取：从当前响应和历史跳转中扫描并解析候选 JWT 或 JWE。
14. 递归 302 追踪：按规范追踪 `Location`，直到终态页面、授权码或终态错误。
15. OAuth Token 交换：仅在授权码和 PKCE 材料完整时执行 token exchange。
16. 终态校验：确认资源状态、令牌来源、失败点、证据链和后续人工复核点。

## 启动模板

开始排障或复现前，先写两张短表，不要边跑边临时猜。

### 状态映射表

把目标实现里的本地状态映射到 skill 状态，至少填这四列：

| 本地状态或函数 | skill 状态 | 进入证据 | 下一跳 |
| --- | --- | --- | --- |
| 例如：返回登录页 / 某个 page.type / 某个回调函数 | `WARMUP` / `CHALLENGE_READY` / `WAITING_MAIL` 等 | 当前响应、日志、cookie 或 URL | 下一次请求或检查点 |

要求：

- 只要代码库里存在多条收尾分支、兜底分支或重登分支，就必须先画这张表。
- 如果某个本地状态已经越过创建动作，但 token 仍未到手，明确标记为 `RESOURCE_COMMITTED` 之后的状态，而不是误判为 `DONE`。

### 最小观测清单

默认至少记录这些字段：

- 会话标识：`session_id`、设备标识、cookie jar 摘要
- 挑战材料：挑战参数、证明材料摘要、对应请求的目标 URL
- OAuth 起点：`auth_url`、`state`、`code_verifier`、`redirect_uri`
- 页面或阶段标识：当前页面类型、阶段名、关键响应字段
- OTP 或外部验证：发送时间、关联键、收到的验证码或验证链接摘要
- 继续链路定位符：租户、工作区、continue URL、302 `Location`
- callback：query、fragment、错误字段、授权码摘要
- token 结果：token exchange 成功或失败原因、当前已拿到的 token 类型

如果这些字段里有任何一个只能从旧缓存拿到，而不是从当前响应或当前 cookie 拿到，显式标记为 `degraded_path=true`。

### 来源证据链

对每个关键字段，再额外记录“它是从哪里来的”，至少使用下面四类来源标签之一：

- `current_response`
  - 来自当前请求的响应体、响应头或 `Location`
- `current_cookie`
  - 来自当前轮 cookie jar 或当前轮请求上下文
- `current_callback`
  - 来自当前轮 callback URL 的 query 或 fragment
- `cached_fallback`
  - 来自旧缓存、旧日志、旧文本或上一轮链路残留

执行规则：

- 只要关键字段的来源是 `cached_fallback`，默认把该字段视为降级恢复，不视为强一致性证据。
- 如果 callback、continue URL、workspace、session token 中任一关键字段只能由 `cached_fallback` 提供，显式记录“跨轮污染风险”。
- 做根因判断时，优先比较“字段值是否一致”，再比较“字段来源是否一致”；值对但来源错，仍然不能判主链健康。

## 实现准则

### 传输层模拟

- 使用单一 `curl_cffi` session 对象承载全流程。
- 在同一闭环中固定一个浏览器 profile，避免中途切换指纹配置。
- 保持请求顺序、来源域、重定向承接和 cookie 演进与真实浏览器一致。
- 对 302、303、307、308 分别按 HTTP 语义处理方法保留或转换，不要偷懒统一成 GET。
- 只在授权要求内使用代理，并把代理信息纳入审计记录。

### 会话强一致性

会话状态至少包括：

- `session_id`
- `cookie_jar`
- `challenge_artifacts`
- `pkce_verifier`
- `pkce_challenge`
- `redirect_chain`
- `mail_correlation_key`
- `committed`
- `jwt_candidates`

要求：

- 任一步骤若需要新建 session，必须视为异常而不是正常分支。
- 任何重试都应从最近一个可恢复检查点继续，而不是丢失前态重新猜测。
- 所有副作用步骤都要带幂等键或可重复识别键。

### 每轮授权请求唯一性规则

把下面这些对象视为“当前轮授权上下文”的一组绑定材料：

- 当前轮 `auth_url`
- 当前轮 `state`
- 当前轮 `code_verifier`
- 当前轮挑战工件
- 当前轮 cookie jar
- 当前轮 callback URL

硬规则：

- 每次进入新的授权请求，都要生成新的 `state` 和 `code_verifier`。
- 任何 challenge 工件只能服务当前轮请求，不能跨轮复用。
- callback 只能由当前轮 `state` / `code_verifier` 解释；如果只能被上一轮解释，直接判为污染或降级。
- 只要当前轮链路主要依赖上一轮缓存继续推进，显式标记 `degraded_path=true`。

### 绑定判定表

用下面的口径判断“是不是还在同一条链路上”：

| 对象 | 保持一致时的信号 | 污染或跨会话信号 |
| --- | --- | --- |
| 会话对象 | 当前请求沿用同一 cookie jar 和连接上下文 | 中途重建 session 后继续复用旧材料 |
| 设备或挑战上下文 | 当前挑战材料与当前设备标识、cookie、预检链对应 | 旧挑战材料被拿到新 session 里继续用 |
| OAuth 上下文 | 当前 callback 能被当前轮 `state` / `code_verifier` 接受 | callback 只能被上一轮或另一轮 OAuth 上下文解释 |
| 继续链路定位符 | 当前租户、工作区、continue URL 来自当前响应 | 只能依赖旧缓存、旧文本或旧重定向结果兜底 |
| 会话令牌 | 能从当前 cookie 或当前响应直接恢复 | 只能从旧日志、旧文本或兜底拼接恢复 |

判定规则：

- 只要关键字段发生来源切换，就不要只写“还能跑通”，而要标记为“链路已降级”。
- 只要当前链路主要靠缓存兜底继续，就不要把它当作强 session 一致性的成功样本。

### 多段 JWT 自动探测解析

自动扫描这些位置：

- `Authorization`、`Set-Cookie`、自定义头
- HTML 正文、JSON 字段、内联脚本
- URL query、fragment、302 `Location`
- 邮件正文和验证链接

解析规则：

- 候选格式优先识别三段 JWS 和五段 JWE。
- 对每一段执行 base64url 安全解码，失败即标记为非 JWT。
- 仅把 payload 作为线索，不把未验证声明当作真实权限依据。
- 记录令牌来源位置、发现步骤、掩码后的头部与声明摘要。

### 递归 302 追踪

- 追踪时保留跳转深度、来源响应、目标 URL、状态码和方法变化。
- 设定最大深度与域名边界，防止无限跳转和跨域漂移。
- 命中终态条件后停止递归：授权码到达、终态页面到达、显式错误到达、深度上限达到。
- 任何跳转中的令牌、验证码、JWT 候选都要在到达终态前先行提取。

## 实战坑型

- 注册成功不等于 token 已到手；服务端可能先创建资源，再要求重登或继续走授权链。
- 当平台已自动发送 OTP 或验证邮件时，不要额外触发发送或重发接口，把重复触发视为潜在破坏动作。
- 继续链路所需的关键定位符可能分散出现在 OTP 校验响应、创建响应、租户或工作区选择响应、cookie jar、`Set-Cookie` 或 302 `Location` 中；把它们当检查点资产缓存，而不是假设只会出现一次。
- 任何预检或挑战请求里的证明材料都必须非空，并与同一轮设备标识、cookie 和挑战上下文配套；不要跨会话借用。
- callback 解析要同时覆盖 query 和 fragment；某些链路会把 `code`、`state` 或错误信息拆在不同位置。

## 风控处理规则

本技能不提供“绕过风控”的可执行做法。正确做法是识别、留痕、止损、升级。

### 识别信号

- 403、429、设备指纹挑战、验证码、JavaScript 挑战、不可解释的跳转环
- 要求真实浏览器证明、WebAuthn、短信二次验证、人工审核页
- 邮件延迟异常、重复验证码失效、同一会话材料突然失效

### 允许的处置

- 降低并发，严格遵守 `Retry-After` 和官方速率限制。
- 切换到用户提供的测试租户、沙箱环境或 allowlist 环境。
- 请求目标系统提供正式测试入口、测试账号或文档化豁免。
- 把阻断点写入证据链，并停止后续会修改资源的自动操作。

### 禁止的处置

- 使用代理池、住宅 IP、设备农场或打码服务规避控制。
- 伪造浏览器证明、篡改设备绑定材料、重放一次性验证码。
- 伪造或修改 JWT 声明、伪造 OAuth 回调、跳过平台要求的人工验证。

## 自愈状态机设计

把流程设计成显式状态机，不要把重试散落在各函数里。

推荐状态：

- `INIT`
- `WARMUP`
- `CHALLENGE_READY`
- `PKCE_READY`
- `IDENTITY_SUBMITTED`
- `WAITING_MAIL`
- `MAIL_PARSED`
- `FLOW_RESUMED`
- `RESOURCE_COMMITTED`
- `JWT_EXTRACTED`
- `REDIRECT_TRACED`
- `TOKEN_EXCHANGED`
- `BLOCKED`
- `FAILED`
- `DONE`

设计要求：

- 每个状态只负责一个可验证目标。
- 每个状态都定义进入条件、成功条件、失败分类、最大重试次数和检查点输出。
- 瞬时失败和永久失败必须分开处理；需要人工接管时进入 `BLOCKED`。
- `RESOURCE_COMMITTED` 是提交屏障。进入该状态后，不允许回到会产生重复创建的上游状态。
- 邮件轮询必须可中断、可续跑、可超时，并保留最近一次成功解析结果。

建议把失败分成三类：

- `transient`：网络抖动、上游超时、邮件暂未到达，可在预算内重试。
- `permanent`：参数错误、授权不足、环境不匹配，立即失败。
- `human_required`：风控阻断、多因素验证、策略页拦截，停止自动化并升级。

## 最小输出要求

使用本技能时，最终产物至少应包含：

- 授权范围与环境假设
- 16 步链路的当前断点
- session 检查点摘要
- 是否已命中 Step 12 提交屏障
- JWT 候选来源与解析摘要
- 302 链路摘要
- token exchange 成功或失败原因
- 风控命中证据与人工接管建议

## 最小验证矩阵

- 挑战路径：确认请求实际发送了非空证明材料，不接受占位值或空字符串。
- OTP 路径：确认自动发送链路下没有额外的发送或重发调用。
- 重登路径：确认“资源已创建，但 token 需二次登录获取”的分支能闭环。
- callback 路径：确认 query、fragment、302 `Location` 三处都能提取授权码、状态或错误。
- 会话路径：确认租户、工作区、会话令牌或同类关键标识可从 cookie、响应体或缓存检查点恢复，而不是仅依赖单一来源。

## 默认执行姿态

- 先求证据，再求推进。
- 先保 session 一致性，再谈重试。
- Step 12 一旦成功即提交，不做侥幸回滚。
- 风控命中即停，不把规避技巧写进实现。
