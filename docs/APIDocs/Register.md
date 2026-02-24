---
sidebar_position: 1
---

# VPN 用户注册接口（Register）

**服务名**：ClyoxVPN  \
**接口版本**：v2.3  \
**维护人**：Auth / Risk Team  \
**最后更新**：2026-02-05

## 1. 接口概览（Overview）

* **Method**：`POST`
* **Path**：`/api/v2/auth/register`
* **Content-Type**：`application/json`  
* **Rate Limit**：5 次/分钟/IP（防止恶意注册）
  * 5 次 / 分钟 / IP
  * 其他频控策略基于综合风控规则执行（不对外暴露）

本接口为 `v2.3` 升级版，主要解决旧版注册接口无法针对不同区域（尤其是 EU 地区）自动触发 GDPR 声明的问题。

* **核心变动**：引入了 `gdpr_consent` 硬性校验及 `X-Experiment` 灰度路由。
* **注意**：对于中东等部分无试用政策区域，后端将返回 ``trialExpiresAt = null``，前端需隐藏试用倒计时 UI。

:::info

说明

* 当前仅基于 IP、注册频率、账号特征 做风控。
* 风控或合规校验失败时，流程立即终止，不会创建任何用户记录。

:::

## 2. 请求头（Headers）

|Header|必填|说明|
|---|---|---|
|`X-Request-Id`|是|请求唯一标识，用于日志与问题排查|
|`X-Client-Version`|否|客户端版本号（用于灰度与兼容判断）|
|`Accept-Language`|否|返回文案语言，默认 `en-US`|

## 3. 请求参数（Body）

```json
{
  "username": "tester_vpn_0",
  "password": "pw123456789",
  "email": "dev_test@clyox.com",
  "phone": "13912345678",
  "promo_code": "WELCOME30",
  "gdpr_consent": true
}
```

**参数说明**

|  字段名   |  类型 | 必填 |   描述 |
| ---|---|---|---|
| ``username``  |  String  | 是  | 用户名（4-20 位，字母或数字）  |
| ``password``  |  String  | 是  | 密码（需包含大小写字母及数字，不少于 8 位）  |
| ``email`` |  String  | 是  | 注册邮箱，用作登录凭证  |
| ``phone`` |  String  | 否  | 手机号（中国大陆 11 位），用于找回/风控  |
| ``promo_code`` |  String  | 否  | 推广码，用于延长试用期  |
| ``gdpr_consent`` |  boolean  | 条件必填  | EU 地区用户必须为 `true`|

**参数校验备注**

* `username` 当前 **不支持下划线和特殊字符**。（历史兼容性原因）
* `email` 需通过标准 RFC 校验。
* `password` 校验在服务端完成，**客户端不要写死规则**。
* 是否属于 EU 用户由 **IP Geo + 账单地区** 综合判断。（IP 可能存在误判）

## 4. 成功返回

```json
{
  "code": 200,
  "message": "Register success",
  "data": {
    "userId": "u_987654321",
    "createdAt": "2026-02-04T10:30:00Z",
    "trialExpiresAt": "2026-02-10T10:30:00Z",
    "configProfileId": "trial_profile_001",
    "nextStep": "NONE"
  },
  "requestId": "req_abc123"
}
```

**返回字段说明**

|  字段名   | 描述 |
| ---|---|
| ``userId``  | 系统生成的唯一用户 ID  |
| ``createdAt``  | 账号创建时间（服务端时间）  |
| ``trialExpiresAt`` | 试用到期时间（未下发则为空） |
| ``configProfileId`` | 试用配置 ID（未下发则为空）  | 
| ``nextStep`` |  后续动作，如 ``VERIFY_EMAIL``  |
| ``requestId`` |  对应请求的 Request Id  |

## 5. 错误码说明

**通用错误结构**

```json
{
  "code": 42910,
  "message": "Too many requests",
  "detail": {
    "retryAfter": 60,
    "rule": "ip_rate_limit"
  },
  "requestId": "req_abc123"
}
```

**错误码列表**

|  HTTP 状态码   |  错误码 | 错误信息 |   场景说明 |
| ---|---|---|---|
| **400**  |  40001  | Invalid parameter| 参数校验失败 |
| **400** | 40002 | Missing gdpr_consent | EU 用户未传 GDPR 同意字段 |  
| **400** | 40003 | Weak password | 密码强度不足（需包含大小写及数字） |
| **409**  |  40901  | Email already exists  | 邮箱已注册 |
| **409** | 40902  | Username already exists  | 用户名冲突|
| **403** |  40310 |Registration blocked| 命中风控规则  |
| **403** |  40320 |GDPR consent required|EU 用户未同意 GDPR  |
| **429** |  42910 |Too many requests| 触发频率限制  |
| **500** |  50000 |Internal server error| 系统异常  |

## 6. 灰度与版本说明（v2.3）

* `v2.3` 功能默认关闭，通过以下条件开启：
  * 客户端 Header：``X-Client-Version >= 2.3``
  * 或服务端用户组白名单
* 灰度内容包含：
  * 注册风控规则调整
  * 试用配置下发策略优化

## 7. 业务处理流程（简要）

1. 基础参数校验
2. 注册频率与风险规则校验
3. 区域合规校验（GDPR / 服务可用性）
4. 创建用户账号
5. 根据地区与灰度策略：
   * 下发试用配置（如允许）
   * 或仅创建账号，不生成配置
