---
sidebar_position: 2
---

# VPN 用户登录接口（Login）

**服务名**：ClyoxVPN  \
**接口版本**：v2.3  \
**维护人**：Auth / Risk Team  \
**最后更新**：2026-02-25

## 1. 接口概览（Overview）

* **Method**：`POST`
* **Path**：`/api/v2/auth/login`
* **Content-Type**：`application/json`
* **Rate Limit**：10 次/分钟/IP（防止撞库与暴力尝试）
  * 超过阈值将触发临时封禁或验证码策略（不对外暴露）

本接口用于已注册用户的账号登录，并在校验通过后返回登录态凭证。
登录流程受 账号状态、区域合规、风控策略、灰度规则 共同影响。

:::info

* 登录失败不会改变账号状态。（如冻结、删除等）
* 连续失败可能触发额外风控校验，但不会直接锁号。

:::

## 2. 接口特性

* **用途**：校验用户凭证并建立登录态（返回 `access token`）。

* **风控**：基于登录频率、IP、账号异常行为判断是否放行；
命中风控时可能返回 403 / 429。

* 合规：

  * 已注册 EU 用户，如账号未完成 GDPR 同意，将拒绝登录。

  * 登录接口 **不会补采 GDPR 同意**，需引导用户回注册 / 设置页处理。

* **灰度**：当 `X-Experiment: auth_v23` 命中时，启用 `v2.3` 登录风控策略。

## 3. 请求头（Headers）

|Header|必填|说明|
|---|---|---|
|`X-Request-Id`|是|请求唯一标识，用于日志与问题排查|
|`X-Client-Version`|否|客户端版本号（用于灰度与兼容判断）|
|`Accept-Language`|否|返回文案语言，默认 `en-US`|

## 4. 请求参数（Body）

```json
{
  "login": "dev_test@clyox.com",
  "password": "pw123456789",
  "remember_me": true
}
```

**参数说明**

|  字段名   |  类型 | 必填 |   描述 |
| ---|---|---|---|
| ``login``  |  String  | 是  | 登录标识（邮箱或用户名）  |
| ``password``  |  String  | 是  | 密码（需包含大小写字母及数字，不少于 8 位）  |
| ``remember_me`` |  boolean  | 否  | 是否延长登录态有效期证  |

**参数校验备注**

* l`ogin` 支持 邮箱或用户名，不支持手机号直接登录。
* 登录接口不会区分“账号不存在”和“密码错误”，统一返回登录失败。
*密码校验始终在服务端完成。

## 5. 成功返回

```json
{
  "code": 200,
  "message": "Login success",
  "data": {
    "userId": "u_987654321",
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresAt": "2026-02-12T12:30:00Z",
    "nextStep": "NONE"
  },
  "requestId": "req_def456"
}
```

**返回字段说明**

|  字段名   | 描述 |
| ---|---|
| ``userId``  | 登录用户唯一 ID  |
| ``accessToken``  | 登录态访问凭证  |
| ``expiresAt`` | Token 过期时间（服务端时间） |
| ``nextStep`` |  后续动作，如 ``VERIFY_EMAIL``  |
| ``requestId`` |  对应请求的 Request Id  |

## 6. 错误码说明

**通用错误结构**

```json
{
  "code": 40301,
  "message": "Login failed",
  "requestId": "req_def456"
}
```

**错误码列表**

|  HTTP 状态码   |  错误码 | 错误信息 |   场景说明 |
| ---|---|---|---|
| **400**  |  40001  | Invalid parameter| 参数校验失败 |
| **401** | 40101 | Invalid credentials | 账号或密码错误 |  
| **403** |  40310 |Login blocked| 命中风控规则  |
| **403** |  40321 |GDPR consent required|EU 用户未同意 GDPR  |
| **429** |  42911 |Too many login attemptss| 登录尝试过于频繁  |
| **500** |  50000 |Internal server error| 系统异常  |

## 7. 灰度与版本说明（v2.3）

* `v2.3` 功能默认关闭，通过以下条件开启：
  * 客户端 Header：``X-Client-Version >= 2.3``
  * 或服务端用户组白名单
* 灰度内容包含：
  * 登录风控阈值调整
  * 异常账号识别策略优化

## 8. 业务处理流程（简要）

1. 参数与格式校验
2. 账号存在性校验
3. 密码校验
4. 风控与合规校验
5. 生成并返回登录态

## 9. 注意事项

* 登录接口 **不会创建账号或补全注册信息**。
* 登录成功不代表账号具备完整服务权限，需以配置接口结果为准。
* 多端同时登录可能导致旧 token 失效（取决于账号策略）。

## 10. 非目标

* 本接口不处理注册流程。
* 不提供密码找回或重置能力。
* 不处理设备绑定与多设备管理。

## 11. 常见误用与注意事项

* 登录失败不等同于账号被封禁。
* 401 错误不建议自动重试。
* EU 用户若被提示 GDPR 问题，应引导至账号设置页面。
* 风控拒绝可能与注册阶段规则不同，请勿复用注册错误处理逻辑。
