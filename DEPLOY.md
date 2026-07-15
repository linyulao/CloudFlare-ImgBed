# 部署到 Cloudflare Workers

本文档介绍两种部署方式：**GitHub Actions 自动部署**（推荐）和 **Wrangler CLI 手动部署**。

> 安全提醒：所有凭据（API Token、Account ID）请通过 Cloudflare dashboard 和 GitHub Secrets 注入，**不要提交到仓库**，**不要贴到任何 AI 工具 / 聊天记录 / issue 中**。

---

## 方式一：GitHub Actions 自动部署（推荐）

部署在 push 到 `main` 分支时自动触发。

### 1. 准备工作

#### 1.1 创建 Cloudflare API Token
1. 打开 https://dash.cloudflare.com/profile/api-tokens
2. 点 **Create Token** → **Custom token**
3. 权限配置（**只勾必需的**，最小权限原则）：

| 资源 | 权限 |
|------|------|
| Account.Workers Scripts | Edit |
| Account.Workers KV Storage | Edit |
| Account.D1 | Edit |
| Account.Account Settings | Read（创建 Token 时默认需要） |

4. **Token 只在创建后显示一次**，立即复制并保存到密码管理器（1Password / Bitwarden / Keychain）。

#### 1.2 获取 Account ID
登录 Cloudflare dashboard，Account ID 显示在右下角或 Workers 页面顶部。

### 2. Fork 仓库并配置 Secrets

1. Fork `Marseventh/CloudFlare-ImgBed` 到你自己的 GitHub 账号
2. 进入 fork 后的仓库 → **Settings** → **Secrets and variables** → **Actions**
3. 添加以下 **Repository secrets**（**New repository secret**）：

| Secret 名 | 是否必填 | 说明 |
|----------|---------|------|
| `CLOUDFLARE_API_TOKEN` | ✅ 必填 | 第 1.1 步创建的 Token |
| `CLOUDFLARE_ACCOUNT_ID` | ✅ 必填 | 第 1.2 步获取的 ID |
| `D1_DATABASE_ID` | ⚠️ 推荐 | 部署前先创建 D1 数据库，把 ID 填入；可留空先部署 |
| `KV_NAMESPACE_ID` | 可选 | 用于 `random/` 随机图接口 |
| `R2_BUCKET_NAME` | 可选 | R2 存储桶名 |
| `WORKER_NAME` | 可选 | 自定义 Worker 名称，默认 `cloudflare-imgbed` |
| `WORKER_VARS` | 可选 | JSON 字符串，业务环境变量（不推荐，部署后在管理面板配更安全） |

> ⚠️ **绝不要把 Token 填到 Variables 里**。Variables 对所有有仓库访问权限的人可见，Secrets 是加密的。

### 3. 触发部署

- **自动触发**：push 到 `main` 分支
- **手动触发**：GitHub → **Actions** → **Deploy to Cloudflare Workers** → **Run workflow**

部署日志在 Actions 页面查看。

### 4. 部署后必须配置（云端）

部署成功 ≠ 可用。Worker 启动后需要：

#### 4.1 绑定存储资源
进入 Cloudflare dashboard → **Workers & Pages** → 选中你的 Worker → **Settings** → **Bindings**：

| 类型 | 变量名 | 说明 |
|-----|-------|------|
| D1 Database | `img_d1` | 推荐，用于存储文件元数据 |
| R2 Bucket | `img_r2` | 用于文件存储（如果用 R2 渠道） |
| KV Namespace | `img_url` | 用于随机图接口（可选） |

> 也可以在第 2 步直接把 ID 填到 Secrets，让 GitHub Action 自动注入到 `wrangler.toml`。
> 两种方式效果一样，区别是 **dashboard 绑定更直观，Secrets 注入便于批量管理**。

#### 4.2 初始化数据库
首次部署需要执行 SQL 迁移：
1. Cloudflare dashboard → **Workers & Pages** → **D1** → 选中数据库
2. 点 **Console** 标签
3. 复制 `database/init.sql` 内容粘贴执行

#### 4.3 访问管理面板
打开 Worker 域名（`https://<worker-name>.<your-subdomain>.workers.dev`），按提示配置：
- 管理员账号密码
- 上传渠道（Telegram / Discord / R2 / S3 / HuggingFace / WebDAV）
- 图床策略（鉴权、压缩、目录等）

#### 4.4 绑定自定义域名（可选）
Worker → **Settings** → **Triggers** → **Custom Domains** → 添加。

---

## 方式二：Wrangler CLI 本地部署

适合需要在本地构建并直接发布的场景。

### 1. 登录
```bash
npx wrangler login
```
浏览器会打开 Cloudflare 授权页面。

### 2. 创建资源（首次部署）
```bash
# D1 数据库（推荐）
npx wrangler d1 create img_d1
# 记下输出的 database_id

# KV 命名空间（可选）
npx wrangler kv namespace create img_url
# 记下输出的 id

# R2 存储桶（可选）
npx wrangler r2 bucket create img_r2
```

### 3. 编辑 wrangler.toml
编辑 `deploy/worker/wrangler.toml`，取消注释并填入：
- `database_id = "..."` （D1）
- `id = "..."` （KV）
- `bucket_name = "..."` （R2）

> **不要把真实的 ID 提交到 Git**。`.gitignore` 应该已经忽略了生成的配置；如果用 GitHub Actions 部署，配置会从 Secrets 注入，不在仓库里持久化。

### 4. 部署
```bash
npx wrangler deploy --config deploy/worker/wrangler.toml
```

部署成功后会输出 Worker URL。

---

## 部署架构

```
┌─────────────────────────────────────────┐
│   Cloudflare Worker (imgbed)            │
│   ├── index.js (入口)                   │
│   ├── functions/* (API 路由)            │
│   ├── ASSETS binding (frontend-dist)    │
│   ├── img_d1 (D1, 元数据)               │
│   ├── img_r2 (R2, 文件)                 │
│   └── img_url (KV, 随机图缓存)          │
└─────────────────────────────────────────┘
              │
              ├──→ Telegram Bot API
              ├──→ Discord Webhook
              ├──→ S3 兼容存储 (AWS / R2 / MinIO)
              ├──→ HuggingFace
              └──→ WebDAV
```

---

## 常见问题

### Q: 部署失败，提示 "Authentication error [code: 10000]"
API Token 权限不够。返回第 1.1 步检查权限。

### Q: 部署成功但访问域名 404
检查 `wrangler.toml` 中 `[assets]` 块的 `directory = "../../frontend-dist"` 路径是否正确（相对于 `deploy/worker/wrangler.toml`）。

### Q: 上传文件失败 "D1 binding not found"
Worker Settings → Bindings → 添加 D1 binding，变量名必须为 `img_d1`。

### Q: 想从 Pages 迁到 Workers
本项目用 `[assets]` 块 + `functions/` 目录模拟 Pages 行为，部署到 Workers 时通过 `wrangler.toml` 的 `[assets]` 绑定 `frontend-dist`。两者行为一致，**直接选 Workers 更简单**。

### Q: 配额超限怎么办
Cloudflare Workers 免费版限制：每天 10 万次请求。可以在 dashboard → Workers → Plans 升级到付费版（$5/月起，含 1000 万次请求）。

---

## 安全建议

1. **Token 最小权限**：只勾选需要的权限，不要给 `Edit All Resources` 之类的全权限 Token
2. **定期轮换**：建议每 3-6 个月重新生成一次 Token
3. **不要泄露**：Token 等同于 Worker 的完全控制权，**绝不提交到 Git、绝不贴到 AI 工具**
4. **监控异常**：Cloudflare dashboard → Account → Audit Logs 可查看 Token 使用记录
5. **绑定泄露凭据**：如果 Token 不慎泄露，立即在 dashboard 撤销（Roll 或 Delete）
