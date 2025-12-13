# 部署指南

本文档说明如何基于本项目搭建自己的代理服务。

## 🚀 推荐方式：Fork + Dashboard（零配置）

**最简单的方式，无需修改任何代码或配置文件！**

### 步骤 1: Fork 项目
1. 点击 GitHub 页面右上角的 **Fork** 按钮
2. Fork 到你自己的 GitHub 账号

### 步骤 2: 部署到 Cloudflare

#### 方式 A: 使用 Cloudflare Pages（推荐）
1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 进入 **Workers & Pages** → **Create application** → **Pages**
3. 点击 **Connect to Git** → 选择你 fork 的仓库
4. 配置构建设置：
   - **Framework preset**: None
   - **Build command**: 留空
   - **Build output directory**: `/`
5. 点击 **Save and Deploy**
6. 部署完成！你会得到一个 `https://xxx.pages.dev` 地址

#### 方式 B: 使用 Cloudflare Workers
1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 进入 **Workers & Pages** → **Create application** → **Workers**
3. 点击 **Create Worker** → 编辑代码
4. 复制 `src/worker.js` 的全部内容，粘贴到编辑器
5. 点击 **Save and Deploy**
6. 部署完成！你会得到一个 `https://xxx.workers.dev` 地址

### 步骤 3: 配置自定义域名（可选）

1. 在 Worker/Page 设置中，找到 **Triggers** 或 **Custom domains**
2. 点击 **Add Custom Domain** 或 **Add route**
3. 输入你的域名（如 `proxy.example.com`）
4. Cloudflare 会自动配置 DNS 和 SSL
5. 完成！

**就这么简单！域名会自动从请求中获取，首页会显示正确的域名。**

---

## 🛠️ 使用 Wrangler CLI（适合开发者）

### 快速部署

```bash
# 1. 克隆项目
git clone <your-fork-url>
cd claudeflareproxy

# 2. 安装 Wrangler
npm install -g wrangler

# 3. 登录
wrangler login

# 4. 部署（无需修改任何配置）
wrangler deploy
```

部署后会得到一个 `https://proxy.your-account.workers.dev` 地址。

### 配置自定义域名

**方式 A: 在 Dashboard 配置（推荐）**
1. 登录 Cloudflare Dashboard
2. 进入你的 Worker → **Settings** → **Triggers** → **Routes**
3. 点击 **Add route**
4. 填写路由：`proxy.example.com/*`
5. 选择你的域名区域
6. 保存

**方式 B: 在 wrangler.toml 配置**
1. 编辑 `wrangler.toml`，取消注释并修改：
   ```toml
   workers_dev = false
   routes = [
     { pattern = "proxy.example.com/*", zone_name = "example.com" }
   ]
   ```
2. 重新部署：`wrangler deploy`

---

## 📊 部署方式对比

| 方式 | 难度 | 需要修改配置 | 适合人群 |
|------|------|-------------|---------|
| Fork + Pages | ⭐ 最简单 | ❌ 不需要 | 所有人 |
| Fork + Workers | ⭐⭐ 简单 | ❌ 不需要 | 所有人 |
| Wrangler CLI | ⭐⭐⭐ 中等 | ❌ 不需要 | 开发者 |
| Dashboard 配置路由 | ⭐⭐ 简单 | ❌ 不需要 | 所有人 |
| wrangler.toml 配置 | ⭐⭐⭐ 中等 | ✅ 需要 | 开发者 |

---

## ⚠️ 注意事项

1. **无需修改代码**：
   - `src/worker.js` 会自动从请求中获取域名
   - 首页会自动显示正确的域名
   - 无需手动配置任何域名

2. **自定义域名**：
   - 域名必须在 Cloudflare 托管
   - DNS 记录会自动创建（如果使用 Dashboard 配置）
   - 推荐在 Dashboard 配置，比修改 wrangler.toml 更简单

3. **推荐流程**：
   ```
   Fork → 连接到 Cloudflare → 部署 → 在 Dashboard 添加自定义域名 → 完成
   ```

---

## 🔧 故障排除

### 问题 1: 部署后无法访问

**解决方案**：
- 等待 1-2 分钟，DNS 传播需要时间
- 清除浏览器缓存
- 使用无痕模式访问

### 问题 2: 自定义域名配置失败

**检查**：
- 域名是否在 Cloudflare 托管
- DNS 记录是否开启代理（橙色云朵）
- 是否有权限管理该域名

### 问题 3: 首页显示的域名不正确

**原因**：可能是缓存问题

**解决方案**：
- 清除浏览器缓存
- 使用无痕模式访问
- 域名会自动从请求中获取，无需手动配置

---

## 📚 更多信息

- [Cloudflare Workers 文档](https://developers.cloudflare.com/workers/)
- [Cloudflare Pages 文档](https://developers.cloudflare.com/pages/)
- [Wrangler CLI 文档](https://developers.cloudflare.com/workers/wrangler/)
- [自定义域名配置](https://developers.cloudflare.com/workers/configuration/routing/routes/)

---

## 💡 最佳实践

1. **新手推荐**：Fork → Cloudflare Pages → Dashboard 配置域名
2. **开发者推荐**：Wrangler CLI 部署 → Dashboard 配置域名
3. **团队协作**：使用 wrangler.toml 配置，代码即配置

**记住：无论哪种方式，都无需修改 `src/worker.js` 代码！**
