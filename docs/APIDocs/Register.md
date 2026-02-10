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
