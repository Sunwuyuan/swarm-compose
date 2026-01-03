# Traefik 配置转换说明

本文档记录了从 OpenResty/Nginx 配置到 Traefik 的转换过程和配置说明。

## 配置文件结构

```
traefik/
├── docker-compose.yml          # Traefik 容器编排配置
└── dynamic/
    ├── routers.yml             # 路由、服务和中间件配置
    └── README.md               # 本文档
```

## 转换映射表

| OpenResty 配置文件 | 域名/IP | 后端地址 | 特殊处理 |
|---|---|---|---|
| 154.37.221.120_17151/proxy/root.conf | 154.37.221.120 | http://10.163.231.1:35740 | Host头指定为目标地址 |
| coderun-1.190823.xyz/proxy/root.conf | coderun-1.190823.xyz | http://10.163.231.8:3033 | HSTS头 |
| cskv.wuyuan.dev/proxy/root.conf | cskv.wuyuan.dev | http://100.74.26.106:3475 | 标准转发 |
| koishi.wuyuan.dev/proxy/root.conf | koishi.wuyuan.dev | http://100.81.13.25:5140 | HSTS头 |
| monitor.wuyuan.dev/proxy/root.conf | monitor.wuyuan.dev | http://100.64.13.72:25774 | HSTS头 |
| scratch.190823.xyz/proxy/proxy.conf | scratch.190823.xyz | http://100.64.13.72:3068 | WebSocket支持、HSTS头 |
| search-1.190823.xyz/proxy/root.conf | search-1.190823.xyz | http://10.163.231.8:7700 | HSTS头 |
| search.wuyuan.dev/proxy/root.conf | search.wuyuan.dev | http://100.64.13.72:8087 | 标准转发 |
| social.wuyuan.dev/proxy/root.conf | social.wuyuan.dev | http://10.163.231.180:3000 | HSTS头 |
| subc/proxy/root.conf | subc | http://100.64.13.72:15051 | 内容替换（需要插件） |
| vw.wuyuan.dev/proxy/root.conf | vw.wuyuan.dev | http://100.64.13.72:5658 | 标准转发 |
| zerocat-api.houlangs.com/proxy/root.conf | zerocat-api.houlangs.com | http://100.64.13.72:3000 | WebSocket支持 |

## OpenResty 配置项转换说明

### 1. location 指令
- **OpenResty**: `location ^~ /`
- **Traefik**: `rule: "Host(...)"`
- 说明：Traefik 使用 Host 规则匹配，自动匹配所有路径

### 2. proxy_pass
- **OpenResty**: `proxy_pass http://127.0.0.1:3000;`
- **Traefik**:
  ```yaml
  services:
    svc-xxx:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:3000"
  ```

### 3. proxy_set_header
- **OpenResty**: `proxy_set_header Host $host;`
- **Traefik**: 由 `forwardAuth` 和 `headers` 中间件处理
  - `X-Real-IP`、`X-Forwarded-For`、`X-Forwarded-Proto` 等自动添加
  - 特殊情况（如154地址）通过自定义中间件设置

### 4. proxy_http_version 1.1
- **Traefik**: 默认使用 HTTP/1.1 向后端转发
- 无需显式配置

### 5. add_header
- **OpenResty**: `add_header X-Cache $upstream_cache_status;`
- **Traefik**: 通过 `headers` 中间件的 `customResponseHeaders` 添加

### 6. Upgrade / WebSocket 支持
- **OpenResty**:
  ```nginx
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  ```
- **Traefik**: 在 `forward-headers-websocket` 中间件中处理，Traefik 原生支持 WebSocket

### 7. HSTS 安全头
- **OpenResty**: `add_header Strict-Transport-Security "max-age=31536000";`
- **Traefik**: 通过 `hsts-header` 中间件添加

### 8. Content Replacement (subc)
- **OpenResty**:
  ```nginx
  sub_filter "127.0.0.1:25500" "154.37.221.120:15052";
  sub_filter_once off;
  sub_filter_types *;
  ```
- **Traefik**: 需要部署 Traefik 插件实现响应体替换
  - 推荐使用: `github.com/traefik/plugin-replace` 或自定义插件
  - 当前配置中作为占位符保留

## 启用 HTTPS

所有路由都配置了：
- `entryPoints: [web, websecure]` - 同时监听 HTTP 和 HTTPS
- `tls.certResolver: le` - 使用 Let's Encrypt 自动获取证书
- `web` 入口点自动重定向到 `websecure`

## 健康检查

所有服务都配置了健康检查：
```yaml
healthCheck:
  path: "/"
  interval: "30s"
  timeout: "5s"
```

可根据实际应用调整路径和间隔。

## 特殊说明

### 154.37.221.120_17151
原 OpenResty 配置中的 Host 头设置为具体的后端地址 `10.163.231.1:35740`，
这在 Traefik 中通过 `forward-headers-154` 中间件实现。

### subc - 内容替换需求
原配置中对响应体进行了替换：
- 将 `127.0.0.1:25500` 替换为 `154.37.221.120:15052`

目前有以下解决方案：
1. **部署 Traefik 插件**: 使用官方或第三方插件实现响应体替换
2. **前端处理**: 通过 JavaScript 在客户端进行替换
3. **API 网关**: 使用额外的反向代理层处理内容替换

## 部署步骤

1. **验证配置**
   ```bash
   cd traefik
   docker-compose config
   ```

2. **启动 Traefik**
   ```bash
   docker-compose up -d
   ```

3. **验证路由**
   ```bash
   curl -v http://localhost/
   # 应该被重定向到 HTTPS
   ```

4. **监控日志**
   ```bash
   docker-compose logs -f
   ```

## 回滚步骤

如果需要回到 OpenResty 配置：
1. 保存当前 Traefik 配置文件（已在 git 中）
2. 恢复 OpenResty 容器部署
3. 重新配置原有反向代理

## Git 提交说明

- 所有 Traefik 配置文件已提交到 `traefik/` 目录
- 原 OpenResty 配置文件保留在 `sites/*/proxy/` 目录中作为参考
- 通过版本控制可以追溯配置历史和进行对比

## 参考资源

- Traefik 官方文档: https://doc.traefik.io
- Traefik 路由规则: https://doc.traefik.io/traefik/routing/routers/
- Traefik 中间件: https://doc.traefik.io/traefik/middlewares/http/overview/
