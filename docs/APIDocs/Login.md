---
sidebar_position: 2
---

# VPN 用户登录接口（Login）

**服务名**：ClyoxVPN  \
**接口版本**：v2.3  \
**维护人**：Auth / Risk Team  \
**最后更新**：2026-02-24

## 1. 接口概览（Overview）

* **Method**：`POST`
* **Path**：`/api/v2/auth/login`
* **Content-Type**：`application/json`

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
