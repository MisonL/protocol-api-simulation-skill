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
- 成功优先判定：一旦 Step 12 的核心资源或核心会话已经被服务端确认提交，立即把该动作视为已提交，后续只做收尾、取证和幂等保护，不再回滚猜测。
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
- 如果任务涉及安全演练、流量录制、重放、隔离实验环境或证据工件管理，再按需读取 `references/security-lab-lessons.md`。
- 只有在任务真的命中这些坑型时才加载参考文件；不要把任何站点的私有行为默认套到别的目标上。

## 规范优先

默认先以协议规范和通用不变量为主，再记录目标站点的偏差行为。

执行规则：

- 先问“这条链路按 OAuth/OIDC/HTTP 语义本应如何工作”，再问“这个站点实际上做了什么”。
- 如果站点行为偏离规范，不要直接改写主状态机定义；先单独记录为“站点偏差”。
- 主 skill 只沉淀通用规律；站点特有参数、页面流转、缓存兜底习惯下沉到 `references/`。
- 当站点偏差和规范冲突时，输出里同时保留“规范预期”和“站点现实”，不要只保留后一者。

## 首轮 5 问

首次使用时，先回答下面 5 个问题，再决定要不要跑满后续闭环：

1. 当前任务的授权范围是什么：域名、账号、租户、允许副作用边界是否明确。
2. 这轮更适合 `observe_only`、`replay_only` 还是 `mutating_run`。
3. 当前轮关键绑定材料是否齐全：`state`、`code_verifier`、cookie jar、challenge 工件、callback。
4. 当前环境是否可复位，还是只能做录制、抓包和证据保留。
5. 这轮最想确认的事实是什么：协议逻辑、传输实现，还是观测器缺失。

如果前 3 问里有任一项答不上来，默认不要进入 `mutating_run`。

### 首次使用最短路径

首次上手时，默认按下面 4 步进入，而不是先通读全文件：

- 先确定当前模式和授权范围。
- 再写状态映射表与最小观测清单。
- 然后只保留当前轮证据，不急着续跑。
- 最后根据绑定一致性与复位条件，决定是停在观测、进入重放，还是阻断。

## 16 步闭环

推荐按下面的闭环组织流程，并为每一步定义输入、输出、失败分类和可重试性：

使用边界：

- 这是分析框架，不要求所有任务都跑满 16 步。
- 对只做被动观测的任务，可在 Step 4 或 Step 14 停止，不必进入创建、token exchange 或提交屏障相关步骤。
- 对“资源或会话已存在，只需补授权或补 token”的续跑任务，可直接从 Step 11 开始，只要当前轮 PKCE、callback 和会话材料完整可证。

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
12. 提交确认：一旦服务端确认核心资源或核心会话已经提交，立即标记 `committed=true`。
13. JWT Payload 提取：从当前响应和历史跳转中扫描并解析候选 JWT 或 JWE。
14. 递归 302 追踪：按规范追踪 `Location`，直到终态页面、授权码或终态错误。
15. OAuth Token 交换：仅在授权码和 PKCE 材料完整时执行 token exchange。
16. 终态校验：确认资源状态、令牌来源、失败点、证据链和后续人工复核点。

说明：

- 在 OAuth 或 OIDC 续跑场景里，Step 12 可能表示“授权会话已提交”而不是“新资源已创建”。
- 进入 Step 12 之后，只允许继续做 token 收尾、证据补齐和幂等确认，不允许回到会再次触发创建或注册的上游步骤。

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
- OTP 或外部验证：`otp_sent_at`、关联键、收到的验证码或验证链接摘要
- 继续链路定位符：租户、工作区、continue URL、302 `Location`
- callback：`callback_raw_url`、query、fragment、错误字段、授权码摘要
- token 结果：token exchange 成功或失败原因、当前已拿到的 token 类型

如果这些字段里有任何一个只能从旧缓存拿到，而不是从当前响应或当前 cookie 拿到，显式标记为 `degraded_path=true`。

执行顺序：

- callback 解析优先看当前轮 callback URL，再看当前轮 302 `Location`，最后才回看历史响应或缓存。
- `continue_url`、`workspace`、`session_token` 优先从当前响应和当前 cookie 获取，其次看当前轮 302 `Location` 或 callback，最后才允许 `cached_fallback`。
- `state` 和 `code_verifier` 只能来自当前轮生成材料，不能从缓存恢复。

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
- `state` 和 `code_verifier` 禁止从 `cached_fallback` 恢复；一旦发生，立即标记 `degraded_path=true` 并停止 token exchange。
- 如果 callback 只能由上一轮缓存或上一轮 OAuth 上下文解释，直接进入降级或阻断状态，不继续假定当前轮链路健康。
- 对 `continue_url`、`workspace`、`session_token` 这类继续链路定位符，优先使用当前轮可验证来源；若多个当前轮来源互相冲突，停止推进并记录 `source_conflict=true`，不要静默择一。
- 对注册/登录续跑场景，默认提取顺序是：`current_response` > `current_cookie` > `current_callback` > `cached_fallback`；若目标字段语义上只应出现在 callback，则单独说明该例外。
- 只要 `degraded_path=true`，最终结论默认只能写弱结论，不宣称“强 session 一致性成功”。
- 只要授权码、continue URL 或会话令牌依赖 `cached_fallback`，默认停止 token exchange，先补当前轮观测。
- 一旦记录 `source_conflict=true`，固定进入 `BLOCKED`，先固化最小证据包，再决定是否重新开一轮授权；不要继续 token exchange 或继续推进当前轮。

### 最小证据包

每次运行至少保留下面四类工件：

- 范围声明
  - 域名、账号、租户、允许副作用范围、当前执行模式。
- 原始工件
  - 抓包流、HAR、302 链、邮件原文摘要、关键请求与响应摘录。
- 归一化状态
  - `state`、`code_verifier`、cookie 摘要、continue URL 来源、callback 摘要。
- 运行结论
  - 当前归因、断点、`degraded_path=true` 与否、是否命中提交屏障。

如果只有最终日志，没有原始工件和来源标签，默认把该结论视为弱证据。

### 执行模式分层

把每次运行明确标成下面三种模式之一，不要混着执行：

- `observe_only`
  - 只做被动观测、录制、抓包、日志对齐，不发送会改变服务端状态的请求。
- `replay_only`
  - 只重放已观察到的请求或最小继续链路，用于验证状态绑定、cookie 演进、302 语义和回放一致性。
  - 最小重放只允许包含证明协议行为所需的最少请求集，禁止引入会产生新状态、删除状态或触发次生操作的请求。
- `mutating_run`
  - 允许创建、更新或触发真实副作用，但前提是目标环境可复位，或用户已明确批准在一次性测试租户中执行。

执行规则：

- 把三种模式视为硬阶段：Phase 1=`observe_only`，Phase 2=`replay_only`，Phase 3=`mutating_run`。
- 默认从 `observe_only` 开始，证据不足时再升级到 `replay_only`，最后才进入 `mutating_run`。
- 输出中必须显式写明当前模式、升级原因和退出条件。
- 授权范围不清、环境不可复位、或存在单用户限制时，不自动进入 `mutating_run`。
- 如果测试实例是共享的、单用户的，或复位能力未知且不可验证，禁止进入任何会修改状态的步骤。
- 只有同时满足“范围已确认”“重放目标已录制”“当前轮关键字段可绑定”“不会引入新增副作用”四个条件时，才允许从 `observe_only` 升到 `replay_only`。
- 只有同时满足“复位方法可验证且已实测或有追溯记录”“实例或租户隔离已证明”“副作用请求具备幂等或可回收识别键”“升级前的最小证据包已落盘”四个条件时，才允许从 `replay_only` 升到 `mutating_run`。
- 若复位方法不可验证，或共享实例/共享账号边界不能独立证明，当前任务仅限观测和录制，不得执行任何 mutating 请求。
- `replay_only` 只允许重放已观测到的原始方法、URL、头部、cookie 和 body；不得补造未观测字段，不得引入新跳转，不得触发额外发送、重发或创建接口。
- 无法证明“副作用可回收”时，默认停在 `observe_only` 或 `replay_only`，不要做 best-effort 续跑。

模式切换闸门：

- `observe_only -> replay_only`
  - 已拿到当前轮证据，且知道要验证哪一个绑定关系或哪一段跳转链。
- `replay_only -> mutating_run`
  - 已确认副作用范围、复位方法、单用户约束和幂等保护。
- 任意模式 -> `BLOCKED`
  - 风控阻断、人工验证、环境不可复位、或关键来源只能依赖 `cached_fallback`。

每次切换模式时，输出里至少补三项：

- 为什么升级或降级
- 当前禁止动作是什么
- 如果失败，如何回到上一个安全检查点

### 实验环境与复位点

进入任何会产生副作用的运行前，先确认下面四件事：

- 隔离边界：测试账号、租户、工作区、实例是否彼此独立。
- 复位方法：如何删除测试资源、回收验证码、清空 cookie、重置邮箱关联键。
- 单用户约束：目标如果天然是单用户或单会话应用，不要让多个测试链路共享同一实例。
- 密钥口径：同一轮实验用到的测试密钥、flag key、回调域或 allowlist 口径必须一致。

如果这些前置条件不成立，把当前任务视为高风险观测任务，只做证据采集，不做自动续跑。复位能力未知时，默认等同于“不可写入环境”。

示例：

- 共享测试实例不等于可续跑；若账号、租户或复位边界不能独立证明，则默认按不可复位处理。
- 不可复位环境默认禁止 `mutating_run`。

### OTP 与重登卫生

当链路包含 OTP、验证邮件或“注册成功后仍需重登”时，默认遵守下面规则：

- 先判断服务端是否已经自动发信，再决定是否允许 resend；不要把 resend 当默认动作。
- 记录 `otp_sent_at`、邮箱关联键、最近一次成功解析结果和对应会话标识。
- 新一轮 OTP 或重登开始前，清理上一轮遗留的 continue URL、workspace、session token 摘要和邮件解析缓存。
- 一旦进入 `RESOURCE_COMMITTED`，后续只允许收尾、重登补 token 或取证，不允许回到会重复创建资源的上游状态。

自动发送判定顺序：

1. 先看 `current_response` 是否明确出现“已发送验证码/验证邮件”或等价阶段迁移。
2. 再看 `current_cookie`、`current_callback` 或当前轮页面状态里是否出现新的邮件关联键、发送时间或验证阶段标识。
3. 再看当前轮邮箱、短信或外部验证通道里是否出现与本轮会话一致的到件证据。
4. 只有前三步都无法证明“已自动发送”，且目标系统允许 resend，才把 resend 视为候选动作。

执行规则：

- 任一当前轮证据已经证明“已自动发送”时，禁止再触发 resend。
- 只要 resend 会引入新的验证码、覆盖旧验证码或改变 continue URL，就把它视为副作用步骤，而不是普通重试。

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

### Callback 判定规则

- 如果 callback 同时包含 query 和 fragment，分别解析两侧的 `code`、`state`、`error`，不要只取一边。
- 如果 query 与 fragment 给出的关键字段互相冲突，把该 callback 视为无效证据，进入 `BLOCKED` 或 `FAILED`，并保留原始输入。
- 如果授权码存在，但当前轮 `state` 或 `code_verifier` 缺失、不匹配或只能由旧缓存解释，禁止 token exchange，进入收尾失败态。
- callback 只能证明“当前轮可继续”，不能自动证明“当前轮已健康闭环”；是否健康仍要看字段来源和会话绑定。

### 多段 JWT 自动探测解析

自动扫描这些位置：

- `Authorization`、`Set-Cookie`、自定义头
- HTML 正文、JSON 字段、内联脚本
- URL query、fragment、302 `Location`
- 邮件正文和验证链接

解析规则：

- 候选格式优先识别三段 JWS 和五段 JWE。
- 对每一段执行 base64url 安全解码，失败即标记为非 JWT。
- 只有当候选字段同时满足“分段数正确”“字符集符合 base64url”“解码后头部或声明呈 JSON 对象”时，才升级为 `candidate_token`。
- 对非标准 base64、随机长字符串或无法满足 JWT/JWE 结构约束的字段，只标记为 `suspected_token`，不自动判真。
- 仅把 payload 作为线索，不把未验证声明当作真实权限依据。
- 记录令牌来源位置、发现步骤、掩码后的头部与声明摘要。
- 对候选按“当前轮 callback > 当前轮响应 > 当前轮 cookie > 旧缓存”排序；旧缓存候选默认最低优先级。
- 对内容相同但来源不同的候选不做无脑去重；来源差异本身也是证据。
- 对每个候选统一输出：`source`、`location`、`token_type`、`mask`、`parse_status`、`payload_summary`、`verification_status`。

### Token 三层分离

把 token 相关工作拆成三层，不要混为一谈：

- 结构层
  - 先判断它是不是 JWS / JWE / 其他 bearer token，能否被安全解码。
- 校验层
  - 再判断签名、issuer、audience、nonce、过期时间或相关校验条件是否成立。
- 权限层
  - 最后才根据已校验的声明或服务端返回结果判断权限、身份和可用性。

执行规则：

- 结构层通过，不代表 token 可信。
- 校验层失败时，权限层默认不能继续给出正向结论。
- 未验签 payload 只能用于排障线索、字段发现和状态定位，不能用于声称“这个 token 有某权限”。
- 如果目标链路同时返回 `id_token`、`access_token`、`refresh_token`，分开记录，不要把它们混成一个“token 已拿到”的笼统状态。

### 递归 302 追踪

- 追踪时保留跳转深度、来源响应、目标 URL、状态码和方法变化。
- 设定最大深度与域名边界，防止无限跳转和跨域漂移。
- 命中终态条件后停止递归：授权码到达、终态页面到达、显式错误到达、深度上限达到。
- 任何跳转中的令牌、验证码、JWT 候选都要在到达终态前先行提取。
- 默认顺序是：先记录当前响应中的候选 token 和 callback 线索，再继续追 302，最后才决定是否 token exchange。

### 升级观测条件

当纯协议重放已经不能解释事实时，升级观测器，而不是继续猜。

优先升级到这些高可信观测手段：

- 真实浏览器基线
  - 用于确认页面真实流转、前端脚本行为、JS challenge、WebAuthn、复杂 cookie 演进。
- 抓包或拦截代理
  - 用于确认真实请求顺序、头部差异、302 链、挑战参数来源、缓存命中与否。
- 对照实验
  - 把“纯 HTTP 模拟”和“真实浏览器基线”并排比较，定位差异出在哪一层。

默认升级条件：

- 当前链路事实主要依赖猜测，而不是当前观测证据。
- challenge、验证码、302 或 callback 的来源无法证明属于当前轮。
- 纯 HTTP 客户端无法解释真实浏览器才能触发的页面状态或脚本行为。
- 多轮重试后仍无法分清问题是在协议逻辑、传输实现还是观测缺失。
- 真实浏览器基线与纯 HTTP 模拟的差异已经出现，但还没有明确标注差异落在协议逻辑、传输实现还是观测器。

命中下面任一项时，升级观测不再是建议，而是必做：

- 真实浏览器与纯 HTTP 的 302 `Location` 不一致
- 任一关键 `Set-Cookie` 只出现在其中一条链路
- callback 的 query、fragment 或错误字段在两条链路间不一致
- token 或 challenge 工件只在其中一条链路中出现

升级后最少比较这些差异维度：

- 302 `Location` 与最终落点
- `Set-Cookie` / 请求 Cookie 的增删变化
- callback query / fragment / 错误字段
- challenge 参数与证明材料来源
- JWT/JWE 候选的出现位置与结构差异

建议直接按下面的固定对照表头记录，避免每次手写 diff：

| 维度 | 真实浏览器基线 | 纯 HTTP 模拟 | 结论 |
| --- | --- | --- | --- |
| 请求方法 + URL | | | |
| 状态码 + `Location` | | | |
| `Set-Cookie` / 请求 Cookie | | | |
| callback 原始 URL | | | |
| challenge 来源 | | | |
| token / JWE / JWT 候选位置 | | | |

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
- `RESUME_AUTH`
- `WAIT_FOR_CALLBACK`
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
- 注册成功后若仍需重登或继续授权，应直接进入 `RESOURCE_COMMITTED` 之后的 `RESUME_AUTH`、`WAIT_FOR_CALLBACK` 或同类后续状态，禁止回到任何会重复创建资源的上游状态。
- `16 步闭环` 是分析框架；这里的状态列表是实现状态机。不要把两者当作重复要求，而应保持一一映射关系。

建议把失败分成三类：

- `transient`：网络抖动、上游超时、邮件暂未到达，可在预算内重试。
- `permanent`：参数错误、授权不足、环境不匹配，立即失败。
- `human_required`：风控阻断、多因素验证、策略页拦截，停止自动化并升级。

## 最小输出要求

最终输出至少覆盖下面这 10 行骨架，避免写成散文：

- mode: `observe_only` / `replay_only` / `mutating_run`
- mode_change_reason: 为什么保持、升级或降级到当前模式
- breakpoint: 当前停在第几步，或哪个 skill 状态
- committed: `true` / `false`
- current_round_bindings: 完整 / 缺失 / 污染
- degraded_path: `true` / `false`
- reset_plan: 复位方法 / 不可复位
- strongest_evidence: 当前最强证据来自哪里
- root_cause_hypothesis: 协议逻辑 / 传输实现 / 观测缺失
- next_safe_action: 下一步最安全动作是什么

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
- callback 如果只能由上一轮缓存解释，立即标记 `degraded_path=true` 并停止 token exchange。
- 风控命中即停，不把规避技巧写进实现。
