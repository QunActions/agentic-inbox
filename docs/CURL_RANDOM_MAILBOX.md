# 使用 cURL 创建随机邮箱并获取邮件

网页地址是 `https://cloudmail.qun.run`。使用固定 API Token 调用接口时，请使用 Worker 原始地址，避免被 Cloudflare Access 的网页登录重定向拦截。

## 前提

应用生产环境启用了 Cloudflare Access。API 请求可以使用 Access JWT，或使用后台配置的固定 API Token：

```bash
export INBOX_URL='https://agentic-inbox.qunwang6.workers.dev'
export CF_ACCESS_JWT='从 Cloudflare Access 登录会话取得的 JWT'
# 后台 API Token（如果已配置）
export API_TOKEN='后台提供的固定 token'
```

请求头写法：

```bash
-H "CF-Access-JWT-Assertion: ${CF_ACCESS_JWT}"
```

使用固定 Token 时改为：

```bash
-H "Authorization: Bearer ${API_TOKEN}"
```

固定 Token 只对 `/api/*` 和 `/mcp` 生效，不能直接打开网页；网页仍需 Cloudflare Access 登录。

如果把 `INBOX_URL` 改成 `https://cloudmail.qun.run`，请求会先被 Access 拦截并返回 `302` 登录跳转；这种情况下请改用 Access JWT，而不是固定 Token。

如果使用 Cloudflare Access Service Token，也可以按 Access 策略要求改用对应的 `CF-Access-Client-Id` 和 `CF-Access-Client-Secret` 请求头。

## 创建随机邮箱

调用随机邮箱接口：

```bash
curl -sS -X POST "${INBOX_URL}/api/v1/mailboxes/random" \
  -H "CF-Access-JWT-Assertion: ${CF_ACCESS_JWT}" \
  -H 'Accept: application/json'
```

成功响应示例：

```json
{
  "id": "7f3a91c2d0@cloudmail.qun.run",
  "email": "7f3a91c2d0@cloudmail.qun.run",
  "name": "Inbox",
  "settings": {}
}
```

保存邮箱地址：

```bash
MAILBOX=$(curl -sS -X POST "${INBOX_URL}/api/v1/mailboxes/random" \
  -H "CF-Access-JWT-Assertion: ${CF_ACCESS_JWT}" | jq -r '.email')
echo "${MAILBOX}"
```

注意：该接口只在 `EMAIL_ADDRESSES` 为空时允许随机创建；如果部署配置了固定白名单，会返回 `403`。

## 获取邮箱列表

```bash
curl -sS "${INBOX_URL}/api/v1/mailboxes" \
  -H "CF-Access-JWT-Assertion: ${CF_ACCESS_JWT}" \
  -H 'Accept: application/json' | jq
```

## 获取收件箱邮件

邮箱地址包含 `@`，建议使用 `--get --data-urlencode` 让 curl 正确编码路径参数：

```bash
MAILBOX='7f3a91c2d0@cloudmail.qun.run'

curl -sS --get "${INBOX_URL}/api/v1/mailboxes/${MAILBOX}/emails" \
  --data-urlencode 'folder=inbox' \
  --data-urlencode 'page=1' \
  --data-urlencode 'limit=50' \
  -H "CF-Access-JWT-Assertion: ${CF_ACCESS_JWT}" \
  -H 'Accept: application/json' | jq
```

响应通常包含 `emails` 和 `totalCount`：

```json
{
  "emails": [],
  "totalCount": 0
}
```

## 获取单封邮件

先从上一步响应中取得邮件 `id`，再请求：

```bash
EMAIL_ID='邮件 id'

curl -sS "${INBOX_URL}/api/v1/mailboxes/${MAILBOX}/emails/${EMAIL_ID}" \
  -H "CF-Access-JWT-Assertion: ${CF_ACCESS_JWT}" \
  -H 'Accept: application/json' | jq
```

## 收信链路

要真正收到外部邮件，还必须在 Cloudflare Email Routing 中将 `cloudmail.qun.run` 的 Catch-all 或具体地址规则指向 `agentic-inbox`。仅创建 API 邮箱不会自动改变 Email Routing。

当前 Worker 对不存在的邮箱地址会忽略来信；因此规则指向 Worker 后，收件地址必须已经在应用中创建。

## 常见错误

- `403 Missing required CF Access JWT`：未带 `CF-Access-JWT-Assertion`，或 Access 尚未登录。
- `403 Invalid or expired Access token`：JWT 已过期，重新登录并获取新 JWT。
- `403 Random mailbox creation is disabled...`：部署配置了 `EMAIL_ADDRESSES` 白名单。
- 邮箱列表为空：检查邮箱地址是否已创建，以及 Email Routing 规则是否指向 `agentic-inbox`。
