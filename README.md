# Cloudflare Workers 反向代理

基于 Cloudflare Workers 的 HTTP/HTTPS 反向代理服务。

## ✨ 特性

- ✅ HTTP/HTTPS 代理转发
- ✅ 支持自定义端口
- ✅ 自动 HTTPS，零维护
- ✅ 全球 CDN 加速
- ✅ 免费部署
- ✅ 自动 CORS 支持
- ✅ **零配置 Fork**：无需修改任何代码

## 🚀 快速开始

### 方法1：Fork + Cloudflare Dashboard（推荐，零配置）

**最简单的方式，无需修改任何代码！**

1. 点击右上角 **Fork** 按钮，Fork 到你的 GitHub 账号
2. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)
3. 进入 **Workers & Pages** → **Create application**
4. 连接你 fork 的 GitHub 仓库
5. 部署完成！
6. （可选）在 Dashboard 添加自定义域名

**就这么简单！域名会自动获取，首页会显示正确的域名。**

### 方法2：Wrangler CLI（适合开发者）

```bash
# 1. 克隆项目
git clone <your-fork-url>
cd claudeflareproxy

# 2. 安装并登录
npm install -g wrangler
wrangler login

# 3. 部署（无需修改任何配置）
wrangler deploy
```

部署后会得到一个 `https://proxy.your-account.workers.dev` 地址。

**⚙️ 配置说明**：
- **无需修改代码**：域名会自动从请求中获取
- **自定义域名**：推荐在 Cloudflare Dashboard 配置，比修改配置文件更简单
- 详细部署指南请查看 [DEPLOY.md](DEPLOY.md)

## 📖 使用方法

### 基础代理

```bash
# HTTPS 代理
curl https://my-proxy.your-account.workers.dev/proxy/httpbin.org/get

# HTTP 代理
curl https://my-proxy.your-account.workers.dev/httpproxy/httpbin.org/get

# 带端口代理
curl https://my-proxy.your-account.workers.dev/proxyport/example.com:8443/api

# 加速 GitHub 仓库克隆
git clone https://my-proxy.your-account.workers.dev/proxy/github.com/QImageLab/cf-proxy.git
```

### 在代码中使用

```javascript
// 代理 API 请求
const response = await fetch('https://my-proxy.your-account.workers.dev/proxy/api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer YOUR_TOKEN'
  },
  body: JSON.stringify({ key: 'value' })
});
```

## 🔗 路由规则

| 路由 | 目标 | 示例 |
|-----|------|------|
| `/proxy/:host/:path*` | `https://:host/:path*` | `/proxy/httpbin.org/get` |
| `/httpproxy/:host/:path*` | `http://:host/:path*` | `/httpproxy/httpbin.org/get` |
| `/proxyport/:host/:port/:path*` | `https://:host::port/:path*` | `/proxyport/localhost:3000/api` |
| `/httpproxyport/:host/:port/:path*` | `http://:host::port/:path*` | `/httpproxyport/localhost:3000/api` |

## 📚 文档
- [⭐ 部署指南 (DEPLOY.md)](DEPLOY.md) - 详细的部署步骤和配置说明
- [⭐ 常见问题 (FAQ.md)](FAQ.md) - 详细的问题解决方案和调试指南

### 🧪 快速测试
```bash
# 测试代理是否支持特定端口
curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net:8200/"

# 批量测试常用端口
for port in 3000 8000 8080 8200; do
    echo "Testing port $port..."
    curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net:$port/" && echo "✅ Port $port works"
done
```

## ⚠️ 已知问题

### ❌ Cloudflare 代理域名的非标准端口访问问题

当代理访问开启了 Cloudflare 代理的域名时，非标准端口（如 8200）会被拒绝。

**解决方案**：
1. 将目标域名的 Cloudflare 代理改为"仅 DNS"（灰色云朵）
2. 使用 Cloudflare 支持的端口（8080, 8443 等）
3. 使用没有 CDN 代理的域名

详细信息请查看 [常见问题 (FAQ)](FAQ.md)。

## 📁 项目结构

```
proxy/
├── README.md          # 项目说明
├── ROUTES.md          # 路由使用指南
├── wrangler.toml      # Workers 配置
└── src/
    └── worker.js      # 核心代码（306行）
```

## ⚠️ 使用限制

- **免费版限制**：10万次请求/天，10ms执行时间
- **适用场景**：API代理、个人使用、小规模应用
- **不适用**：大文件传输（>100MB）、高并发场景

## 🔒 安全建议

- 仅个人使用，勿公开分享
- 不记录任何请求内容
- 仅转发请求，不做修改

## 📄 许可证

MIT License

---

*Made with Cloudflare Workers*