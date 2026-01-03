# Swarm Compose - Docker Swarm 配置库

Docker Swarm 集群的完整配置管理，包括 Traefik 反向代理、监控系统和站点配置。

## 目录结构

```
swarm-compose/
├── traefik/                    # Traefik 反向代理配置
│   ├── docker-compose.yml      # Traefik 服务定义
│   └── dynamic/                # 动态配置文件
│       ├── routers.yml         # 路由、服务和中间件配置
│       └── README.md           # 配置转换说明
├── monitoring/                 # 监控系统配置
│   ├── docker-compose.yml      # Prometheus、Node Exporter、cAdvisor
│   └── prometheus.yml          # Prometheus 抓取配置
├── sites/                      # 站点配置（参考）
│   └── */proxy/                # 原 OpenResty 配置（已转换为 Traefik）
└── README.md                   # 本文件
```

## 主要特性

### Traefik 反向代理
- ✅ 全自动 HTTPS（Let's Encrypt + Cloudflare DNS-01）
- ✅ 支持 12 个站点的路由和负载均衡
- ✅ WebSocket 支持
- ✅ 健康检查
- ✅ 标准的代理头部转发（X-Forwarded-* 等）

### 监控系统
- 📊 Prometheus 指标采集
- 📈 Node Exporter（宿主机监控）
- 🐳 cAdvisor（容器监控）
- 📉 Traefik 自身指标

### 配置管理
- 🔄 所有配置都在版本控制中
- 📝 从 OpenResty 配置完整转换到 Traefik
- 🔐 使用 Docker 密钥管理敏感信息

## 快速开始

### 前置条件
- Docker & Docker Swarm 已初始化
- Cloudflare API Token（用于 DNS-01 验证）
- NFS 服务器（用于存储 ACME 证书）

### 部署步骤

1. **创建必需的 Docker 秘密**
   ```bash
   echo "your-cloudflare-api-token" | docker secret create cf_dns_api_token -
   ```

2. **创建 Traefik 网络**
   ```bash
   docker network create --driver overlay traefik-net
   ```

3. **部署 Traefik**
   ```bash
   cd traefik
   docker stack deploy -c docker-compose.yml traefik
   ```

4. **部署监控系统**
   ```bash
   cd monitoring
   docker stack deploy -c docker-compose.yml monitoring
   ```

5. **验证部署**
   ```bash
   docker stack services traefik
   docker stack services monitoring
   ```

## 配置说明

### Traefik 配置文件
详见 [traefik/dynamic/README.md](traefik/dynamic/README.md)

包含：
- 12 个虚拟主机的路由规则
- 反向代理头部设置
- HSTS 安全头配置
- WebSocket 支持
- 健康检查设置

### 监控配置
Prometheus 监控以下目标：
- Prometheus 自身
- Node Exporter（所有节点）
- cAdvisor（所有节点）
- Traefik（指标采集）

访问地址：`https://prometheus.swarm.wuyuan.dev`（需认证）

## 配置转换历史

原始配置使用 OpenResty/Nginx，配置文件位于 `sites/*/proxy/` 目录。
所有配置已转换为 Traefik YAML 格式，保存在 `traefik/dynamic/routers.yml`。

**转换说明**：
- `location ^~ /` → `rule: "Host(...)"`
- `proxy_pass` → `services.*.loadBalancer.servers`
- `proxy_set_header` → 通过中间件处理
- `add_header` → `middlewares.*.headers.customResponseHeaders`
- WebSocket 支持通过 `forward-headers-websocket` 中间件
- HSTS 头通过 `hsts-header` 中间件

详见 [traefik/dynamic/README.md](traefik/dynamic/README.md#openresty-配置项转换说明)

## 管理命令

### 查看 Traefik Dashboard
```bash
# Dashboard: https://traefik.swarm.192325.xyz
# 用户名: admin
# 密码: 见 docker-compose.yml 中的 basicauth
```

### 查看日志
```bash
# Traefik 日志
docker service logs traefik_traefik

# Prometheus 日志
docker service logs monitoring_prometheus
```

### 更新配置
1. 编辑 `traefik/dynamic/routers.yml`
2. Traefik 自动重新加载配置（--providers.file.watch=true）
3. 验证日志中没有错误

### 新增路由
1. 在 `routers.yml` 中添加新的路由配置
2. 保存文件，Traefik 自动应用
3. 验证 Dashboard 中显示新路由

## 故障排查

### 证书获取失败
```bash
# 检查 ACME 日志
docker service logs traefik_traefik | grep acme

# 验证 Cloudflare 密钥
docker secret ls
```

### 路由不工作
```bash
# 检查 Traefik 日志
docker service logs traefik_traefik

# 验证配置文件语法
docker exec <traefik-container> cat /etc/traefik/dynamic/routers.yml
```

### 健康检查失败
```bash
# 检查后端服务是否运行
docker ps | grep <service-name>

# 测试后端连接
docker exec <traefik-container> curl http://<backend-ip>:<port>/
```

## 版本控制

所有配置都通过 Git 管理，支持：
- 配置历史追溯
- 变更审计
- 回滚操作

主要分支：
- `main` - 生产配置

## 安全建议

1. 定期更新 Docker 镜像
2. 通过 Cloudflare 保护 DNS
3. 定期备份 ACME 证书（NFS）
4. 使用 Docker 秘密管理凭证
5. 限制 Dashboard 访问（已配置 Basic Auth）

## 贡献指南

1. 创建新分支进行修改
2. 提交变更前验证配置
3. 提交 Pull Request
4. 在 `main` 分支部署后需要手动更新实际集群

## 许可证

内部使用
