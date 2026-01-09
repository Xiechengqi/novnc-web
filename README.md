# noVNC Web Client

HTML5 VNC 客户端，支持通过浏览器远程访问桌面。

## HTTPS 域名代理配置

当通过 HTTPS 域名代理访问 noVNC 时，需要正确配置反向代理以支持 WebSocket 连接。

### 工作原理

noVNC 客户端通过以下逻辑确定 WebSocket 连接参数：

```
协议: https:// 访问时自动使用 wss://，http:// 访问时使用 ws://
主机: 默认使用当前页面的 hostname
端口: 默认使用当前页面的端口（HTTPS 为 443，HTTP 为 80）
路径: 默认为 websockify
```

最终 WebSocket 连接 URI 格式：`wss://<host>:<port>/websockify`

### 配置要求

1. **反向代理**：需要将 `/websockify` 路径代理到后端 WebSocket 服务
2. **WebSocket 服务**：需要运行 `websockify` 将 WebSocket 协议转换为 VNC 协议
3. **WebSocket 升级**：反向代理必须支持 WebSocket 协议升级

### Nginx 配置示例

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # 静态文件服务
    location / {
        root /path/to/novnc-web;
        index vnc.html;
    }

    # WebSocket 代理配置
    location /websockify {
        proxy_pass http://127.0.0.1:6080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }
}
```

### Caddy 配置示例

```caddy
your-domain.com {
    root * /path/to/novnc-web
    file_server

    reverse_proxy /websockify localhost:6080
}
```

### Apache 配置示例

```apache
<VirtualHost *:443>
    ServerName your-domain.com

    SSLEngine on
    SSLCertificateFile /path/to/cert.pem
    SSLCertificateKeyFile /path/to/key.pem

    DocumentRoot /path/to/novnc-web

    # 启用 WebSocket 代理模块
    # 需要: a2enmod proxy proxy_http proxy_wstunnel

    ProxyPass /websockify ws://127.0.0.1:6080/
    ProxyPassReverse /websockify ws://127.0.0.1:6080/
</VirtualHost>
```

### 启动 websockify 服务

websockify 用于将 WebSocket 连接转换为 VNC 协议：

```bash
# 基本用法：监听 6080 端口，转发到 VNC 服务器 5900 端口
websockify 6080 localhost:5900

# 带 SSL 证书（如果不使用反向代理的 SSL 终止）
websockify --cert=/path/to/cert.pem --key=/path/to/key.pem 6080 localhost:5900

# 后台运行
websockify -D 6080 localhost:5900
```

### URL 参数

可通过 URL 参数覆盖默认配置：

| 参数 | 说明 | 示例 |
|------|------|------|
| `host` | VNC 主机地址 | `?host=192.168.1.100` |
| `port` | WebSocket 端口 | `?port=6080` |
| `path` | WebSocket 路径 | `?path=websockify` |
| `encrypt` | 使用 wss 加密 | `?encrypt=1` |
| `password` | VNC 密码 | `?password=secret` |
| `view_only` | 仅查看模式 | `?view_only=1` |

完整示例：
```
https://your-domain.com/vnc.html?host=vnc-server&port=443&path=websockify&encrypt=1
```

### 故障排查

1. **检查浏览器控制台**：打开 F12 开发者工具，查看 Console 和 Network 面板中的 WebSocket 连接状态

2. **验证 WebSocket 路径**：确认反向代理正确配置了 `/websockify` 路径

3. **检查 websockify 服务**：确认 websockify 服务正在运行并监听正确的端口

4. **混合内容问题**：HTTPS 页面必须使用 wss:// 协议，不能使用 ws://

5. **防火墙设置**：确保相关端口已开放
