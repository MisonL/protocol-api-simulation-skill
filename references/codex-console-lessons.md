# codex-console Lessons

在任务命中下面任一场景时读取本文件：

- Sentinel PoW 或浏览器指纹挑战
- 注册成功后仍需重登或继续授权链
- OTP 自动发送与重复发送冲突
- `continue_url`、`workspace_id`、`session_token` 的多来源提取
- callback URL 可能混用 query 和 fragment

## 1. Sentinel 不是装饰字段

`codex-console` 把 Sentinel 当作真实门槛，而不是固定占位参数。

- 参考实现：<https://github.com/dou-jiang/codex-console/blob/main/src/core/openai/sentinel.py>
- 请求发送点：<https://github.com/dou-jiang/codex-console/blob/main/src/core/http_client.py>
- 回归测试：<https://github.com/dou-jiang/codex-console/blob/main/tests/test_registration_engine.py>

可迁移经验：

- 发送挑战请求前生成非空 `p` 令牌。
- 让 Sentinel 计算依附同一轮 `user-agent` 和会话上下文。
- 用行为测试断言“真的发了非空 PoW”，不要只断言接口 200。

## 2. 注册成功不等于授权闭环

`codex-console` 的注册链路明确覆盖“先注册成功，再重登拿 token”的情况。

- 参考实现：<https://github.com/dou-jiang/codex-console/blob/main/src/core/register.py>
- OAuth 处理：<https://github.com/dou-jiang/codex-console/blob/main/src/core/openai/oauth.py>
- 回归测试：<https://github.com/dou-jiang/codex-console/blob/main/tests/test_registration_engine.py>

可迁移经验：

- 把“资源创建成功”和“token 已获取”拆成两个状态。
- Step 12 之后允许进入收尾或重登分支，但不允许回到会重复创建资源的上游状态。
- 对“需要重登补 token”的路径做单独验证，不要默认注册响应里一定直接带 token。

## 3. OTP 是状态机，不是一次函数调用

`codex-console` 的测试显式防止重复发送 OTP，并记录 OTP 发送时刻。

- 自动发送与重登分支测试：<https://github.com/dou-jiang/codex-console/blob/main/tests/test_registration_engine.py>
- 主流程状态记录：<https://github.com/dou-jiang/codex-console/blob/main/src/core/register.py>

可迁移经验：

- 先判断服务端是否已自动发信，再决定是否允许 resend。
- 把 `otp_sent_at`、邮箱关联键、最近一次成功解析结果纳入检查点。
- 每一轮 OTP 开始前清理旧缓存，避免旧 `continue_url` 或旧 workspace 污染新阶段。

## 4. continue_url、workspace 和 session token 都要多来源提取

`codex-console` 没有假设这些值只会在一个位置出现。

- continue/workspace 缓存与回退：<https://github.com/dou-jiang/codex-console/blob/main/src/core/register.py>
- cookie 中 workspace/session 提取：<https://github.com/dou-jiang/codex-console/blob/main/src/core/register.py>

可迁移经验：

- 优先从当前响应获取；失败后再回退到缓存检查点。
- 从响应体、cookie jar、`Set-Cookie`、请求 cookie 文本和重定向链多处扫描。
- 把“当前值”和“缓存值”都记入审计输出，避免误以为数据来自单一真相源。

## 5. callback 解析必须兼容脏输入

`oauth.py` 对 callback URL 做了较强的兼容解析。

- 参考实现：<https://github.com/dou-jiang/codex-console/blob/main/src/core/openai/oauth.py>

可迁移经验：

- 同时解析 query 与 fragment，并在 query 缺失时用 fragment 补齐。
- 对只有 `?code=...`、裸 query、带 fragment 的半结构化回调都做兼容处理。
- 在 token exchange 前强校验 `state`，并把错误回调视为显式失败，不要静默继续。
