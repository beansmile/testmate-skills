# 身份校验（`/testmate setup-auth`）/ Identity Verification

让 TestMate 信任反馈提交者身份：宿主应用后端用项目的 Signing Secret 签发短期 JWT，widget 通过 `userTokenFn` 获取并随请求发送（`X-User-Token`）。

## 安全边界 / Security Boundary

- Signing Secret 只能存放在宿主应用**服务端**的密钥管理中。绝不写进源码、浏览器 JS、HTML、日志、聊天，或客户端可见的环境变量（如 `NEXT_PUBLIC_*`）。
- 不要让管理员把 secret 粘贴到聊天里；由账号管理员直接写入目标应用的密钥管理。
- JWT 必须包含 `sub` 和较短的 `exp`；有 `email`/`name` 时一并加入。服务端只信任签名内的声明，不会用未签名的 widget `user` 数据补全。
- 优先用 `userTokenFn`（每次请求前取新 token）；静态 `userToken` 仅适用于短生命周期的服务端渲染页面。
- 不要为验证身份而提交测试反馈；用请求拦截/接口测试验证，不要最终提交报告。

## 1. 验证访问并确定项目

```bash
makethisbetter project list --json
```

`makethisbetter info` 只显示本地状态，不能证明已登录。认证失败则运行 `/testmate setup`（Token 登录：`makethisbetter login --token <token> --api-url https://testmate.beansmile.ai/api/v1`）。按单一项目或域名匹配确定 handle，仍有歧义才询问。

账号管理员可读取项目的 `signing_secret`，但**不要**打印完整响应或提取出的值。先选定一个具体的、有文档的、能从 stdin 写入目标应用服务端密钥存储且不回显的命令；然后在**同一个** `set -o pipefail` shell 中构造三段管道：

1. `makethisbetter project show <handle> --json`
2. `jq -er '.signing_secret // error("Account admin access required")'`
3. 该密钥管理写入命令（必须拒绝空 stdin）

不要执行占位符，不要单独运行前两段。完成后用密钥管理的 metadata/status 命令确认键存在，但不读取值。不要用命令替换、临时明文文件或命令行参数（会暴露在进程列表中）。目标密钥管理没有不回显的写入方式时，停止并请管理员直接配置。字段不存在说明当前用户不是账号管理员，不要绕过该权限。不要使用 MCP `project_show`。

## 2. 添加已认证的 token 接口

检测后端及其现有会话认证，添加对应的完整接口。保留现有认证体系，不引入第二套登录。接口必须：拒绝未登录请求、返回 `{ "token": "..." }`、设置 `Cache-Control: no-store`。

### Rails

```ruby
# Gemfile
gem "jwt"

# config/routes.rb
namespace :api do
  get "testmate/token", to: "testmate_tokens#show"
end

# app/controllers/api/testmate_tokens_controller.rb
class Api::TestmateTokensController < ApplicationController
  before_action :authenticate_user!

  def show
    payload = {sub: current_user.id.to_s, exp: 1.hour.from_now.to_i}
    payload[:email] = current_user.email if current_user.respond_to?(:email) && current_user.email.present?
    payload[:name] = current_user.name if current_user.respond_to?(:name) && current_user.name.present?

    signing_secret = Rails.application.credentials.dig(:testmate, :signing_secret)
    raise KeyError, "TestMate signing secret is not configured" if signing_secret.blank?

    response.headers["Cache-Control"] = "no-store"
    render json: {token: JWT.encode(payload, signing_secret, "HS256")}
  end
end
```

### Express

```js
// npm i jsonwebtoken
const jwt = require("jsonwebtoken")

function requireAuthenticatedUser(req, res, next) {
  if (!req.user) return res.sendStatus(401)
  next()
}

app.get("/api/testmate/token", requireAuthenticatedUser, (req, res) => {
  const signingSecret = process.env.TESTMATE_SIGNING_SECRET
  if (!signingSecret) return res.sendStatus(500)

  const payload = {
    sub: String(req.user.id),
    ...(req.user.email ? { email: req.user.email } : {}),
    ...(req.user.name ? { name: req.user.name } : {}),
  }
  const token = jwt.sign(payload, signingSecret, { algorithm: "HS256", expiresIn: "1h" })
  res.set("Cache-Control", "no-store").json({ token })
})
```

### Next.js App Router + NextAuth

```ts
// npm i jsonwebtoken
// app/api/testmate/token/route.ts
import jwt from "jsonwebtoken"
import { auth } from "@/auth"

export async function GET() {
  const session = await auth()
  if (!session?.user?.id) return new Response(null, { status: 401 })

  const signingSecret = process.env.TESTMATE_SIGNING_SECRET
  if (!signingSecret) return new Response(null, { status: 500 })

  const payload = {
    sub: String(session.user.id),
    ...(session.user.email ? { email: session.user.email } : {}),
    ...(session.user.name ? { name: session.user.name } : {}),
  }
  const token = jwt.sign(payload, signingSecret, { algorithm: "HS256", expiresIn: "1h" })
  return Response.json({ token }, { headers: { "Cache-Control": "no-store" } })
}
```

### Python + Flask-Login

```python
# pip install pyjwt
from datetime import datetime, timedelta, timezone
from flask import current_app, jsonify
from flask_login import current_user, login_required
import jwt

@app.get("/api/testmate/token")
@login_required
def testmate_token():
    payload = {
        "sub": str(current_user.get_id()),
        "exp": datetime.now(timezone.utc) + timedelta(hours=1),
    }
    if getattr(current_user, "email", None):
        payload["email"] = current_user.email
    if getattr(current_user, "name", None):
        payload["name"] = current_user.name

    token = jwt.encode(payload, current_app.config["TESTMATE_SIGNING_SECRET"], algorithm="HS256")
    response = jsonify(token=token)
    response.headers["Cache-Control"] = "no-store"
    return response
```

按应用现有的用户模型调整认证钩子与 `email`/`name` 读取方式，不要编造值。同一宿主账号的 `sub` 保持稳定。接口只返回签名后的 JWT，绝不返回 Signing Secret。

## 3. 初始化 widget

```js
MakeThisBetter.init({
  projectKey: "mtb_proj_xxx",
  apiUrl: "https://testmate.beansmile.ai/api/v1",
  userTokenFn: async () => {
    const response = await fetch("/api/testmate/token", {
      credentials: "same-origin",
      headers: { Accept: "application/json" },
    })
    if (!response.ok) throw new Error("Unable to load TestMate user token")
    const { token } = await response.json()
    return token
  },
})
```

静态 token（仅限服务端渲染、页面会话不会超过 token 有效期）：

```js
MakeThisBetter.init({
  projectKey: "mtb_proj_xxx",
  apiUrl: "https://testmate.beansmile.ai/api/v1",
  userToken: token,
})
```

## 4. 验证 / Verify

1. 在开发/测试环境以登录用户调用接口，本地解码 JWT，确认 `sub`、`exp` 及可用的 `email`/`name`；不要输出 Signing Secret。
2. 未登录请求被拒绝；登录请求返回 `Cache-Control: no-store`。
3. 本地初始化 widget，检查或拦截其下一次发往 `https://testmate.beansmile.ai/api/v1` 的请求，确认带有 `X-User-Token`。在提交反馈之前停止。
4. 确认过期 token 被拒绝，且 `userTokenFn` 能取得新 token（用请求测试或本地拦截，不要提交真实反馈）。

---

### English summary

Store the project Signing Secret only in the host app's server-side secret
manager (pipe `project show --json | jq -er .signing_secret | <non-echoing writer>`
in one `pipefail` shell; never print it, never use MCP `project_show`). Add an
authenticated `/api/testmate/token` endpoint (Rails, Express, Next.js, Flask
examples above) returning a short-lived HS256 JWT with `sub`, `exp`, and
optional `email`/`name`, with `Cache-Control: no-store`. Initialize the widget
with `apiUrl: "https://testmate.beansmile.ai/api/v1"` and `userTokenFn`. Verify
via local interception of `X-User-Token`; do not submit feedback.
