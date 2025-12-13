# 常见问题解答 (FAQ)

本文档记录了项目使用过程中遇到的问题及其解决方案。

---

## ❌ 问题1: Cloudflare Workers 代理非标准端口时被重定向到80端口

**发现日期**: 2025-11-20
**最后更新**: 2025-12-07

### 问题描述
当通过Worker代理访问开启了Cloudflare Proxy的域名的非标准端口（如8200）时，请求会被重定向到80端口或直接失败。

### 具体表现
- ❌ `https://your-proxy.workers.dev/httpproxyport/api.example.com/8200` → 失败（返回521错误）
- ✅ `https://your-proxy.workers.dev/httpproxyport/portquiz.net/8200` → 正常工作
- ❌ `https://your-proxy.workers.dev/httpproxyport/1.2.3.4/8200` → Error 1003 Direct IP access not allowed

### 根本原因分析

#### 1. 关键发现：域名是否使用Cloudflare代理

通过测试发现两个域名的差异：

**`api.example.com`**：
- 主站访问返回 502 Bad Gateway（表明有CDN层）
- 8200端口直连访问正常，返回 `Server: uvicorn`
- 通过代理访问8200端口失败

**`portquiz.net`**：
- 没有使用Cloudflare代理
- 直接返回源站IP，可以访问任意端口
- 666和8200端口都能正常工作

#### 2. Cloudflare CDN 端口限制
当域名开启 Cloudflare 代理模式（橙色云朵）时：
- Cloudflare CDN 只支持特定端口
- **HTTP端口**: 80, 8080, 8880, 2052, 2082, 2086, 2095
- **HTTPS端口**: 443, 2053, 2083, 2087, 2096, 8443
- 非标准端口（如8200, 666等）会被拒绝或重定向

#### 3. IP地址访问限制
- Cloudflare Workers 不允许直接访问IP地址（Error 1003）
- 这是安全策略，防止恶意扫描和攻击

### 技术原理

#### Cloudflare代理工作流程
```
用户请求 → Cloudflare CDN → 源站
          ↑
          └── 只支持特定端口
```

#### 代理失败流程
```
用户 → Workers → Cloudflare CDN → 拒绝非标准端口
                 ↑
                 └── 8200端口不在支持列表
```

#### 成功案例流程
```
用户 → Workers → 直连源站（无CDN） → 任意端口
                 ↑
                 └── portquiz.net没有CDN层
```

### 解决方案

#### 方案1: 关闭Cloudflare代理模式（推荐）
1. 登录 Cloudflare Dashboard
2. 找到域名 `api.example.com`
3. 将DNS记录的代理模式改为"仅 DNS"（灰色云朵）
4. 等待DNS传播（通常5分钟内）

#### 方案2: 使用Cloudflare支持的端口（推荐）
将服务部署在Cloudflare支持的端口上：
```bash
# HTTP: 使用8080或8880
https://your-proxy.workers.dev/httpproxyport/api.example.com/8080/

# HTTPS: 使用8443
https://your-proxy.workers.dev/proxyport/api.example.com/8443/
```

#### 方案3: 使用子域名或新域名
创建一个不经过Cloudflare代理的子域名：
```bash
# 创建 direct.api.example.com（灰色云朵）
https://your-proxy.workers.dev/httpproxyport/direct.api.example.com/8200/
```

#### 方案4: 使用其他无CDN代理的域名
选择没有使用Cloudflare代理的域名来部署服务。

### 验证方法

#### 检查域名是否使用Cloudflare代理
```bash
# 方法1: 查看响应头
curl -I "https://your-domain.com/"
# 如果看到以下内容说明使用了Cloudflare：
# Server: cloudflare
# CF-RAY: xxxxxxxxxxxx-HKG

# 方法2: DNS查询
nslookup your-domain.com
# 如果返回的是Cloudflare IP段，说明启用了代理
```

#### 测试端口可用性
```bash
# 直连测试（绕过代理）
curl -v "http://api.example.com:8200/"

# 通过代理测试
curl -v "https://your-proxy.workers.dev/httpproxyport/api.example.com/8200/"
```

### 相关文档
- [Cloudflare 支持的网络端口](https://developers.cloudflare.com/fundamentals/get-started/reference/network-ports/)
- [Cloudflare 5xx 错误排查](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/)
- [Workers Runtime 环境](https://developers.cloudflare.com/workers/learning/how-workers-works/)

---

## ❓ 问题2: 如何判断哪些域名可以通过代理访问非标准端口？

### 快速判断方法
1. **检查是否有Cloudflare代理**
   - 访问主站，查看响应头是否有 `Server: cloudflare`
   - 或使用在线工具检查

2. **测试域名连通性**
   ```bash
   # 测试主站
   curl -I "https://domain.com/"

   # 测试非标准端口
   curl -I "http://domain.com:8200/"
   ```

3. **推荐使用的测试域名**
   - **`portquiz.net`** - ⭐ 专用端口测试服务
     - ✅ 支持所有端口（1-65535）
     - ✅ 无Cloudflare CDN代理
     - ✅ 直接返回源站响应
     - ✅ 专门设计用于端口连通性测试
   - `httpbin.org` - HTTP测试服务（仅支持标准端口）
   - 自己的服务器（确保无CDN代理）

#### portquiz.net 详解

**为什么选择 portquiz.net？**
- 专门用于端口连通性测试
- 不经过任何CDN或代理层
- 直接返回测试结果：`Port test successful! Your IP: xxx.xxx.xxx.xxx`

**测试任意端口的完整示例**：
```bash
# 1. 直接测试（验证端口本身是否可访问）
curl "http://portquiz.net:8200/"
curl "http://portquiz.net:3000/"
curl "http://portquiz.net:9999/"
curl "http://portquiz.net:666/"   # 任意端口都支持

# 2. 通过代理测试（验证代理是否支持该端口类型）
curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net/8200/"
curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net:3000/"
curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net:666/"
```

**实用技巧**：
```bash
# 批量测试常用端口
for port in 3000 8000 8080 8200 8888 9000; do
    echo "Testing port $port..."
    curl "http://portquiz.net:$port/" && echo "✅ Port $port accessible"
done

# 通过代理测试常用端口
for port in 3000 8000 8080 8200 8888 9000; do
    echo "Testing proxy port $port..."
    curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net/$port/" && echo "✅ Proxy supports port $port"
done
```

**技术原理**：
- portquiz.net 直接解析到源站IP（无CDN层）
- 服务器监听所有端口（1-65535）
- 不受Cloudflare端口限制影响
- 是测试代理端口支持能力的理想工具

#### portquiz.net 端口测试示例
```bash
# 测试任意端口（portquiz.net支持所有端口）
curl "http://portquiz.net:8200/"
curl "http://portquiz.net:3000/"
curl "http://portquiz.net:9999/"

# 通过代理测试任意端口
curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net/8200/"
curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net:3000/"
```

**为什么portquiz.net可以访问任意端口？**
- portquiz.net 没有使用 Cloudflare CDN 代理
- 直接解析到源站IP，不受CDN端口限制
- 专门设计用于端口连通性测试

### 判断流程图
```
域名检查 → 有Cloudflare代理？ → 是 → 只能使用支持端口
                      ↓
                    否 → 可以使用任意端口
```

---

## 🔧 问题3: 本地开发正常，部署后为什么失败？

### 常见原因
1. **本地网络环境不同**
   - 本地可能可以直连目标服务器
   - Cloudflare Workers 有网络策略限制

2. **DNS解析差异**
   - 本地DNS可能直接解析到源站IP
   - Workers环境可能解析到CDN IP

3. **防火墙和安全策略**
   - 目标服务器可能阻止了Cloudflare的IP段
   - Workers有出站连接限制

### 调试步骤
1. 在Worker代码中添加详细日志
2. 使用简单的测试域名验证
3. 逐步排查网络路径

### 本地vs生产环境对比

| 环境特点 | 本地开发 | Cloudflare Workers |
|---------|---------|-------------------|
| 网络路径 | 直连 | 通过Cloudflare网络 |
| DNS解析 | 本地DNS | Workers DNS |
| IP限制 | 无 | 有安全策略 |
| 端口限制 | 无 | 有CDN限制 |

---

## 📝 问题4: 为什么有些端口可以访问，有些不行？

### Cloudflare支持的端口列表

#### HTTP端口
- 80 (标准HTTP)
- 8080 (常见备用HTTP)
- 8880
- 2052
- 2082
- 2086
- 2095

#### HTTPS端口
- 443 (标准HTTPS)
- 8443 (常见备用HTTPS)
- 2053
- 2083
- 2087
- 2096

### 端口测试示例
```bash
# ✅ 这些端口通常可以工作
/proxyport/domain.com:80/
/proxyport/domain.com:443/
/proxyport/domain.com:8080/
/proxyport/domain.com:8443/

# ❌ 这些端口会被拒绝（如果域名有Cloudflare代理）
/proxyport/domain.com:3000/
/proxyport/domain.com:8200/
/proxyport/domain.com:9000/
```

---

## 🚀 问题5: 如何选择合适的部署方案？

### 方案对比表

| 方案 | 优点 | 缺点 | 适用场景 |
|-----|------|------|---------|
| 关闭Cloudflare代理 | 可用任意端口 | 失去CDN保护 | 内部服务，测试环境 |
| 使用支持端口 | 保持CDN保护 | 端口选择受限 | 公开服务，生产环境 |
| 使用子域名 | 灵活配置 | 需要额外配置 | 混合部署 |
| 更换域名 | 简单直接 | 可能影响现有服务 | 新项目 |

### 推荐决策流程
1. **服务是否需要公开访问？**
   - 是 → 使用支持端口
   - 否 → 关闭代理

2. **是否已有固定端口？**
   - 是 → 创建无代理子域名
   - 否 → 改用支持端口

3. **是否需要高可用性？**
   - 是 → 保持CDN保护
   - 否 → 灵活配置

---

## 📚 问题6: 相关技术资源

### 官方文档
- [Cloudflare Workers 文档](https://developers.cloudflare.com/workers/)
- [Cloudflare 网络端口文档](https://developers.cloudflare.com/fundamentals/get-started/reference/network-ports/)
- [Workers 限制和配额](https://developers.cloudflare.com/workers/platform/limits/)

### 社区资源
- [Cloudflare 社区论坛](https://community.cloudflare.com/)
- [Stack Overflow - cloudflare-workers 标签](https://stackoverflow.com/questions/tagged/cloudflare-workers)

### 调试工具
- [Cloudflare 请求跟踪](https://dash.cloudflare.com/?to=/analytics/traces)
- [ Workers 日志查看](https://dash.cloudflare.com/?to=/workers/logs)

### 🧪 问题7: portquiz.net - 端口测试的黄金标准

**portquiz.net** 是测试端口连通性的专用服务，在代理调试中扮演重要角色。

#### 核心特性
- ✅ **全端口支持**: 1-65535 所有端口均可测试
- ✅ **无CDN干扰**: 直接连接源站，无中间层
- ✅ **即时响应**: 快速返回测试结果
- ✅ **免费使用**: 无需注册，无限制调用

#### 为什么portquiz.net可以访问任意端口？

1. **无代理架构**
   ```
   用户 → portquiz.net源站（直接连接）
   ```
   vs

   ```
   用户 → Cloudflare CDN → 源站（端口受限）
   ```

2. **技术实现**
   - 服务器程序监听所有TCP端口
   - 每个端口返回相同的响应格式
   - DNS直接解析到源站IP地址

3. **响应示例**
   ```http
   HTTP/1.1 200 OK
   Content-Type: text/html
   Connection: close

   Port test successful!
   Your IP: 203.0.113.42
   ```

#### 实际应用场景

##### 场景1: 验证代理端口支持能力
```bash
# 测试代理是否支持3000端口
curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net:3000/"

# 如果成功，说明代理和目标都不限制3000端口
# 如果失败，说明某一方限制了该端口
```

##### 场景2: 排查网络问题
```bash
# 步骤1: 直连测试
curl "http://portquiz.net:8200/"
# → 成功？端口8200本身可访问

# 步骤2: 代理测试
curl "https://your-proxy.workers.dev/httpproxyport/portquiz.net:8200/"
# → 成功？代理支持8200端口
# → 失败？代理或目标有限制

# 步骤3: 测试实际目标
curl "https://your-proxy.workers.dev/httpproxyport/your-domain.com:8200/"
# → 对比结果，定位问题
```

##### 场景3: 批量端口测试
```bash
# 测试常用开发端口
ports=(3000 8000 8080 8200 8888 9000 5432 3306)

echo "=== 直连测试 ==="
for port in "${ports[@]}"; do
    result=$(curl -s "http://portquiz.net:$port/" | grep -o "successful" || echo "failed")
    echo "Port $port: $result"
done

echo -e "\n=== 代理测试 ==="
for port in "${ports[@]}"; do
    result=$(curl -s "https://your-proxy.workers.dev/httpproxyport/portquiz.net/$port/" | grep -o "successful" || echo "failed")
    echo "Proxy port $port: $result"
done
```

#### 对比其他测试方法

| 测试方法 | 端口范围 | CDN影响 | 可靠性 | 推荐度 |
|---------|---------|---------|--------|--------|
| portquiz.net | 1-65535 | 无 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| httpbin.org | 80, 443 | 有 | ⭐⭐⭐ | ⭐⭐ |
| 自建服务 | 取决于配置 | 取决于配置 | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| 在线工具 | 有限 | 有 | ⭐⭐ | ⭐ |

#### 最佳实践

1. **调试流程**
   ```
   疑问：代理是否支持某端口？
   ↓
   测试：portquiz.net:端口
   ↓
   成功 → 代理支持该端口类型
   失败 → 代理或环境有限制
   ```

2. **验证步骤**
   ```bash
   # 1. 验证端口可达性
   curl "http://portquiz.net:PORT/"

   # 2. 验证代理支持
   curl "https://proxy/httpproxyport/portquiz.net:PORT/"

   # 3. 测试实际目标
   curl "https://proxy/httpproxyport/target:PORT/"
   ```

3. **故障排除**
   - portquiz.net成功，其他失败 → 目标域名有CDN限制
   - portquiz.net失败 → 端口被网络层阻止
   - 代理测试失败 → 代理配置或环境限制

#### 高级用法

##### 自定义响应解析
```bash
# 提取IP地址
curl -s "http://portquiz.net:8080/" | grep -oP 'Your IP: \K[\d.]+'

# 检查响应时间
curl -w "@curl-format.txt" -o /dev/null -s "http://portquiz.net:3000/"
```

##### 集成到CI/CD
```yaml
# GitHub Actions 示例
- name: Test proxy ports
  run: |
    for port in 3000 8000 8080; do
      if curl -f "https://your-proxy.workers.dev/httpproxyport/portquiz.net/$port/"; then
        echo "✅ Port $port works"
      else
        echo "❌ Port $port failed"
        exit 1
      fi
    done
```

---

**总结**: portquiz.net 是端口测试的"瑞士军刀"，简单、可靠、无限制，是调试代理端口问题的首选工具。

---

## 🛠️ 问题8: 调试工具和实用脚本

### 完整的端口测试脚本

```bash
#!/bin/bash
# proxy-port-test.sh - 代理端口测试工具

PROXY_URL="https://your-proxy.workers.dev"
TEST_DOMAIN="portquiz.net"
PORTS=(3000 8000 8080 8200 8888 9000 5432 3306 6379 27017)

echo "=== 代理端口测试工具 ==="
echo "代理地址: $PROXY_URL"
echo "测试域名: $TEST_DOMAIN"
echo "测试端口: ${PORTS[*]}"
echo

# 颜色定义
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# 测试函数
test_port() {
    local port=$1
    local method=$2

    if [ "$method" = "direct" ]; then
        result=$(curl -s --max-time 5 "http://$TEST_DOMAIN:$port/" | grep -o "successful" || echo "failed")
        echo -n "直连端口 $port: "
    else
        result=$(curl -s --max-time 5 "$PROXY_URL/httpproxyport/$TEST_DOMAIN/$port/" | grep -o "successful" || echo "failed")
        echo -n "代理端口 $port: "
    fi

    if [ "$result" = "successful" ]; then
        echo -e "${GREEN}✅ 成功${NC}"
        return 0
    else
        echo -e "${RED}❌ 失败${NC}"
        return 1
    fi
}

# 执行测试
echo -e "${YELLOW}=== 直连测试 ===${NC}"
direct_success=0
for port in "${PORTS[@]}"; do
    test_port $port "direct" && ((direct_success++))
done

echo -e "\n${YELLOW}=== 代理测试 ===${NC}"
proxy_success=0
for port in "${PORTS[@]}"; do
    test_port $port "proxy" && ((proxy_success++))
done

# 统计结果
echo -e "\n${YELLOW}=== 测试结果 ===${NC}"
echo "直连成功: $direct_success/${#PORTS[@]}"
echo "代理成功: $proxy_success/${#PORTS[@]}"

if [ $proxy_success -eq $direct_success ]; then
    echo -e "${GREEN}✅ 代理支持所有可访问的端口${NC}"
else
    echo -e "${RED}❌ 代理存在端口限制${NC}"
    echo -e "${YELLOW}建议：检查目标域名是否使用了CDN代理${NC}"
fi
```

### 快速诊断命令

```bash
# 一键诊断命令
alias proxy-diag='curl -s "http://portquiz.net:8080/" && echo "直连正常" && curl -s "https://your-proxy.workers.dev/httpproxyport/portquiz.net:8080/" && echo "代理正常" || echo "存在问题"'

# 批量测试特定端口
test-proxy-ports() {
    for port in "$@"; do
        echo "Testing port $port..."
        if curl -f -s "https://your-proxy.workers.dev/httpproxyport/portquiz.net/$port/" > /dev/null; then
            echo "✅ Port $port works"
        else
            echo "❌ Port $port failed"
        fi
    done
}

# 使用示例：test-proxy-ports 3000 8000 8200
```

### 监控脚本

```python
#!/usr/bin/env python3
# proxy-monitor.py - 代理服务监控

import requests
import time
import json
from datetime import datetime

PROXY_URL = "https://your-proxy.workers.dev"
TEST_PORTS = [3000, 8000, 8080, 8200, 8888, 9000]

def test_proxy_port(port):
    """测试代理端口连通性"""
    url = f"{PROXY_URL}/httpproxyport/portquiz.net/{port}/"
    try:
        response = requests.get(url, timeout=10)
        return response.status_code == 200 and "successful" in response.text
    except Exception as e:
        print(f"测试端口 {port} 时出错: {e}")
        return False

def monitor_proxy(interval=300):  # 5分钟检查一次
    """持续监控代理状态"""
    while True:
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        results = {}

        for port in TEST_PORTS:
            results[port] = test_proxy_port(port)
            time.sleep(1)  # 避免请求过快

        success_count = sum(results.values())
        total_count = len(results)

        print(f"\n[{timestamp}] 代理状态: {success_count}/{total_count} 端口正常")

        for port, status in results.items():
            status_str = "✅" if status else "❌"
            print(f"  端口 {port}: {status_str}")

        if success_count == 0:
            # 发送告警（可集成到通知系统）
            print("⚠️ 所有端口都无法访问，代理可能存在问题！")

        time.sleep(interval)

if __name__ == "__main__":
    print("启动代理监控...")
    print(f"监控间隔: 5分钟")
    print(f"监控端口: {TEST_PORTS}")
    monitor_proxy()
```

---

**文档维护**: 如遇到新问题，请及时更新本文档。
**最后更新**: 2025-12-07
**贡献者**: Claude AI Assistant & 用户讨论