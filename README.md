# 🔐 2FA 安全管理系统

一个基于 Cloudflare Workers 的现代化双因素认证 (2FA) 管理系统，支持两种登录模式：

- **模式 A — Cloudflare Access 邮箱 OTP**（推荐，零外部依赖）
- **模式 B — LinuxDO OAuth 标准授权**（原始方式，需 LinuxDO 账号）

---

## ✨ 特性

### 🛡️ 安全
- **登录模式 A**：Cloudflare Access 邮箱 OTP 全域保护 + Worker 内 email 白名单双重防护
- **登录模式 B**：标准 OAuth 2.0 授权登录（LinuxDO 等）
- **端到端加密**：所有敏感数据使用 AES-GCM 加密存储
- **JWT 会话管理**：2 小时自动过期
- **速率限制 + 安全审计日志**

### 📱 2FA 管理
- 多种添加方式（手动输入 / 扫码 / 上传图片 / 批量导入）
- TOTP 6/8 位验证码，30/60 秒周期
- 智能账户分类 + 实时倒计时 + 一键复制

### ☁️ 云端备份
- WebDAV 自动备份（Nextcloud / ownCloud / TeraCloud）
- 加密备份文件 + 按日期自动组织 + 历史管理

### 📥📤 数据迁移
- 支持 JSON / 2FAS / 纯文本格式
- 加密导出 + 批量去重

---

## 📋 部署前置条件

| 项 | 必需 | 说明 |
| --- | --- | --- |
| Cloudflare 账号 | ✅ | 免费版即可 |
| 一个域名（已托管在 Cloudflare） | ✅ | 用于自定义域名绑定，例如 `2fa.yourdomain.com` |
| Node.js ≥ 18 + npm | ✅ | 用于安装 Wrangler CLI |
| GitHub 账号 | 可选 | 用于 fork 本仓库 |
| LinuxDO 账号 | 仅模式 B | 用于创建 OAuth 应用 |

---

## 🚀 部署步骤

### 步骤 1：克隆仓库

```bash
git clone https://github.com/cshdotcom/2fauth.git
cd 2fauth
```

### 步骤 2：安装 Wrangler

```bash
npm install -g wrangler
wrangler --version  # 应显示 ≥ 3.0
```

### 步骤 3：登录 Cloudflare

```bash
wrangler login
# 浏览器会弹出 Cloudflare 授权页，点击 Allow
```

### 步骤 4：创建 KV 命名空间

```bash
wrangler kv:namespace create "USER_DATA"
# 记下返回的 id，下一步要用
```

返回示例：
```json
{ "id": "abcd1234efgh5678..." }
```

### 步骤 5：创建 `wrangler.toml`

仓库提供了 `wrangler.toml.example` 模板。复制为 `wrangler.toml` 并替换占位符：

```bash
cp wrangler.toml.example wrangler.toml
```

编辑 `wrangler.toml`，替换以下占位符：
- `REPLACE_WITH_YOUR_KV_NAMESPACE_ID` → 步骤 4 创建的 KV ID
- `you@example.com,another@example.com` → 你的邮箱白名单（逗号分隔）
- `https://2fa.yourdomain.com` → 你的实际域名
- `REPLACE_WITH_YOUR_TURNSTILE_SITE_KEY` → 步骤 6 创建的 Turnstile site key

### 步骤 6：创建 Cloudflare Turnstile widget

清空账号操作需要二次验证（防误删），使用 CF 自家的 Turnstile（免费、无感验证）。

1. 打开 Cloudflare Dashboard → Turnstile → **Add widget**
2. 填写：
   - **Widget name**: `2fauth-clear-all`
   - **Domain**: `2fa.yourdomain.com`（你的应用域名）
   - **Mode**: `Managed`（推荐，正常用户无感通过）
3. 创建后会得到两个值：
   - **Site Key**：填到 `wrangler.toml` 的 `TURNSTILE_SITE_KEY`
   - **Secret Key**：用 `wrangler secret put` 设置（下一步）

### 步骤 7：设置密钥环境变量

```bash
# 主 JWT 密钥（用于签发 2fauth 会话）
wrangler secret put JWT_SECRET
# 粘贴一个 64 字符的随机 hex，例如：openssl rand -hex 32

# 数据加密密钥（用于 AES-GCM 加密 2FA 数据）
wrangler secret put ENCRYPTION_KEY
# 同样：openssl rand -hex 32

# Turnstile 服务端密钥（步骤 6 创建 widget 时显示的 Secret Key）
wrangler secret put TURNSTILE_SECRET_KEY
```

⚠️ **务必妥善保管 `ENCRYPTION_KEY`**。一旦丢失，已加密的 2FA 数据将无法恢复。

### 步骤 8：部署 Worker

```bash
wrangler deploy
```

部署成功后会输出：
```
Published 2fauth (1.23 sec)
  https://2fauth.<your-subdomain>.workers.dev
```

### 步骤 9：绑定自定义域名

1. 打开 Cloudflare Dashboard → 你的域名 → Workers Routes → Add route
2. 或者：Workers & Pages → `2fauth` → Triggers → Custom Domains → Add Custom Domain
3. 填入 `2fa.yourdomain.com`，保存
4. Cloudflare 会自动创建 DNS 记录并签发 SSL 证书

至此两种模式共有的步骤完成，下面根据你选择的登录模式继续。

---

## 🅰️ 模式 A：Cloudflare Access 邮箱 OTP（推荐）

整套架构：用户访问 `2fa.yourdomain.com` → 自动跳转到 CF Access 邮箱 OTP 登录页 → 输入邮箱 → 收到 6 位 OTP → 输入验证码 → 跳回 2FAuth → 点击「第三方授权登录」按钮 → Worker 直接读 CF Access JWT 签发会话 → 进入主界面。

**优点**：
- 零外部依赖，不需要 LinuxDO 或 GitHub OAuth 应用
- Cloudflare 全球边缘节点处理 OTP 邮件，1-3 分钟内送达
- 整个域名被 Access 保护，未登录用户无法访问任何路径
- 邮箱白名单双层校验（Access + Worker）

### 步骤 A1：启用 Cloudflare Zero Trust

1. 打开 https://one.dash.cloudflare.com
2. 选择你的账户，首次使用会引导你创建一个 Zero Trust 团队
3. 团队名随意，例如 `myteam`，最终得到 `myteam.cloudflareaccess.com`

### 步骤 A2：创建 Access Application

1. Zero Trust Dashboard → Access → Applications → **Add an application**
2. 选择 **Self-hosted**
3. 填写：
   - **Application name**: `2FAuth App`
   - **Session Duration**: `15 minutes`（推荐，见步骤 A4 详解）
   - **Application domain**: `2fa.yourdomain.com`（不要加路径，保护整个域名）
4. 点击 Next 进入 Policy 配置

### 步骤 A3：创建邮箱白名单 Policy

1. **Policy name**: `Allow authorized emails`
2. **Action**: `Allow`
3. **Include** 区域：添加规则
   - Selector: `Emails`
   - Operator: `is`
   - Value: `you@example.com`
4. 点 **Add include** 重复添加其他邮箱（每个邮箱一条规则）
5. 保存 Policy，再保存 Application

### 步骤 A4：将会话超时设为 15 分钟（推荐，增加安全性）

默认 CF Access 会话 24 小时，建议改为 15 分钟，到期后自动要求重新做 OTP 验证。

1. 在 Application 详情页 → **Settings**（或 General settings）
2. 找到 **Session Duration**，改为 `15 minutes`
3. 保存

> 2fauth Worker 内部的 JWT 也会同步 15 分钟过期。两个时间一致避免会话状态混乱。
> 如果你希望长时间不用重新登录，可以改为 `8 hours` 或更长，但会降低安全性。

### 步骤 A5：验证 Access 生效

打开浏览器访问 `https://2fa.yourdomain.com/`，应该被 302 跳转到 `https://<team>.cloudflareaccess.com/cdn-cgi/access/login/...` 页面。

输入白名单中的邮箱 → 收到 OTP → 输入 → 跳回 2FAuth 登录页。

### 步骤 A6：完成登录

1. 在 2FAuth 登录页点击「第三方授权登录」按钮
2. Worker 读取请求头 `Cf-Access-Jwt-Assertion`，解码出 email
3. 校验 email ∈ `ALLOWED_EMAILS`（双重防护）
4. 签发 2fauth 自己的 JWT，写入浏览器 localStorage
5. 自动跳转到主界面

🎉 部署完成！

---

## 🅱️ 模式 B：LinuxDO OAuth 授权登录（原始方式）

如果你已有 LinuxDO 账号并希望用它登录，使用此模式。需要先 fork 原始版本（不含 CF Access patch），或者修改本仓库的 `_worker.js` 恢复 OAuth 调用。

### 步骤 B1：在 LinuxDO 创建 OAuth 应用

1. 登录 https://linux.do
2. 打开 https://linux.do/u/me/preferences/account → **OAuth 应用** 或直接访问 https://linux.do/oauth/applications
3. 点击 **New Application**
4. 填写：
   - **Application name**: `2FAuth`
   - **Redirect URI**: `https://2fa.yourdomain.com/api/oauth/callback`
   - **Confidential**: ✅ 勾选
   - **Scopes**: 默认（read + write）
5. 提交后记录：
   - **Client ID**（App ID，例如 `abc123def456...`）
   - **Client Secret**（点 Reveal 显示）

### 步骤 B2：获取你的 LinuxDO 用户 ID

```bash
# 用你的 LinuxDO 用户名替换 <username>
curl -s "https://linux.do/u/<username>.json" | python3 -c "import sys,json; print(json.load(sys.stdin)['user']['id'])"
```

### 步骤 B3：配置 Worker 环境变量

修改 `wrangler.toml` 的 `[vars]` 段，加入：

```toml
[vars]
OAUTH_BASE_URL = "https://connect.linux.do"
OAUTH_REDIRECT_URI = "https://2fa.yourdomain.com/api/oauth/callback"
OAUTH_ID = "你的 LinuxDO 用户 ID（数字）"
ALLOWED_ORIGINS = "https://2fa.yourdomain.com"
```

> LinuxDO 的 OAuth 端点是 `https://connect.linux.do`，路径为标准的 `/oauth2/authorize`、`/oauth2/token`、`/api/user`，与本仓库 `_worker.js` 完全兼容。

设置 secret：

```bash
wrangler secret put OAUTH_CLIENT_ID
# 粘贴 LinuxDO OAuth Client ID

wrangler secret put OAUTH_CLIENT_SECRET
# 粘贴 LinuxDO OAuth Client Secret

wrangler secret put JWT_SECRET
# openssl rand -hex 32

wrangler secret put ENCRYPTION_KEY
# openssl rand -hex 32
```

### 步骤 B4：重新部署

```bash
wrangler deploy
```

### 步骤 B5：完成登录

1. 浏览器访问 `https://2fa.yourdomain.com/`
2. 点击「第三方授权登录」按钮
3. 跳转到 LinuxDO 授权页，点 Approve
4. 跳回 2FAuth，自动登录

🎉 部署完成！

---

## 📚 环境变量参考

### 模式 A（CF Access）需要的变量

| 变量名 | 类型 | 必需 | 示例 / 说明 |
| --- | --- | --- | --- |
| `USER_DATA` | KV binding | ✅ | 绑定到步骤 4 创建的 KV 命名空间 |
| `JWT_SECRET` | secret | ✅ | `openssl rand -hex 32`（签发 2fauth 会话 JWT，15 分钟过期） |
| `ENCRYPTION_KEY` | secret | ✅ | `openssl rand -hex 32`（AES-GCM 加密 2FA 数据，务必备份） |
| `TURNSTILE_SITE_KEY` | var | ✅ | Turnstile widget site key（公开值，可放 toml） |
| `TURNSTILE_SECRET_KEY` | secret | ✅ | Turnstile widget secret key（服务端校验用） |
| `OAUTH_ID` | var | ✅ | `2fauth-user`（统一用户 ID，所有邮箱共享数据） |
| `ALLOWED_EMAILS` | var | ✅ | 逗号分隔的邮箱白名单（与 Access Policy 一致） |
| `ALLOWED_ORIGINS` | var | ✅ | `https://2fa.yourdomain.com` |
| `OAUTH_REDIRECT_URI` | var | ✅ | `https://2fa.yourdomain.com/api/oauth/callback`（占位，CF Access 模式不实际使用） |

### 模式 B（LinuxDO OAuth）需要的变量

| 变量名 | 类型 | 必需 | 示例 / 说明 |
| --- | --- | --- | --- |
| `USER_DATA` | KV binding | ✅ | 同上 |
| `JWT_SECRET` | secret | ✅ | 同上 |
| `ENCRYPTION_KEY` | secret | ✅ | 同上 |
| `OAUTH_BASE_URL` | var | ✅ | `https://connect.linux.do` |
| `OAUTH_REDIRECT_URI` | var | ✅ | `https://2fa.yourdomain.com/api/oauth/callback` |
| `OAUTH_ID` | var | ✅ | 你的 LinuxDO 数字用户 ID |
| `OAUTH_CLIENT_ID` | secret | ✅ | LinuxDO OAuth Client ID |
| `OAUTH_CLIENT_SECRET` | secret | ✅ | LinuxDO OAuth Client Secret |
| `ALLOWED_ORIGINS` | var | ✅ | `https://2fa.yourdomain.com` |

---

## 🛠️ 管理操作

### 添加/删除登录邮箱（模式 A）

需要同步修改两处：

1. **Cloudflare Access Policy**：
   - Zero Trust Dashboard → Access → Applications → `2FAuth App` → Policies
   - 编辑 `Allow authorized emails` → Add/Remove include rule

2. **Worker 的 `ALLOWED_EMAILS` 变量**（双保险）：
   - Dashboard → Workers & Pages → `2fauth` → Settings → Variables
   - 修改 `ALLOWED_EMAILS` 值（逗号分隔）

### 查看实时日志

```bash
wrangler tail 2fauth
```

### 更新代码

修改 `_worker.js` 后：
```bash
wrangler deploy
```

---

## 🔒 安全建议

1. **妥善保管 `JWT_SECRET` 和 `ENCRYPTION_KEY`**
   - `ENCRYPTION_KEY` 丢失 = 2FA 数据无法恢复
   - 建议同时备份到密码管理器

2. **不要在公开仓库提交 `wrangler.toml` 中的密钥**
   - 真正的密钥用 `wrangler secret put` 设置，不在 toml 里
   - `wrangler.toml` 只放非敏感配置（域名、KV ID、邮箱白名单等）

3. **定期备份**
   - 每周通过 WebDAV 创建加密备份

4. **模式 A 比 B 更安全**
   - 模式 A 整个域名在 Access 后面，未登录用户无法访问任何 API
   - 模式 B 的 API 接口在 OAuth 登录前是公开的（虽然需要 JWT 才能调用）

---

## 🩺 故障排查

| 现象 | 排查方向 |
| --- | --- |
| 打开页面一直跳 Access 登录页 | 浏览器禁用了 cookie，允许 `2fa.yourdomain.com` 的 cookie |
| 输入邮箱提示不在白名单 | 邮箱未加入 Access Policy include，或 `ALLOWED_EMAILS` 未同步 |
| OTP 收不到 | 检查邮箱垃圾箱；CF Access OTP 通常 1-3 分钟送达 |
| 模式 A 点登录后提示 `CF Access JWT missing` | Access 应用配置错误，确保 domain 是 `2fa.yourdomain.com`（不含路径） |
| 模式 A 提示 `Email xxx is not allowed` | Worker 的 `ALLOWED_EMAILS` 未包含当前邮箱 |
| 模式 B 登录后提示 `Unauthorized user` | `OAUTH_ID` 配置错误，应该填你的 LinuxDO 数字用户 ID |
| 模式 B 提示 `Token exchange failed` | `OAUTH_CLIENT_SECRET` 错误，或 LinuxDO OAuth 应用已禁用 |
| API 返回 401 | 2fauth JWT 过期（2 小时），重新点登录按钮续期 |
| 加密数据无法解密 | `ENCRYPTION_KEY` 被更改过；恢复原 key |

---

## 📞 支持

- 🐛 Bug 报告：[GitHub Issues](https://github.com/cshdotcom/2fauth/issues)
- 💬 讨论：[GitHub Discussions](https://github.com/cshdotcom/2fauth/discussions)

如果这个项目对您有帮助，请给我们一个 ⭐ Star！

---

## 📄 许可证

MIT License — 详见 [LICENSE](LICENSE)
