# QQPLAY 夜间模式修复指南

> 适用平台：macOS（Apple Silicon / Intel）+ Android  
> 原理：通过 DNS 劫持将 HTTP 主题请求重定向到本地 Nginx 反代，转发为 HTTPS，绕过国际版不兼容 302 跳转的问题。

---

## 背景

QQPLAY（Google Play 版）切换夜间模式时会请求以下域名的资源：

```
showv6.gtimg.cn
iv6.gtimg.cn
gxh.material.qq.com
qzonestyle.gtimg.cn
```

这些资源的 URL 已从 HTTP 升级为 HTTPS，但国际版客户端不兼容服务端的 302 跳转，导致主题下载失败，报错"主题切换失败，请重试"。

**解决思路**：在本地搭建 DNS + Nginx，将上述域名解析到本机，由 Nginx 将请求转发为 HTTPS 再返回给客户端。

---

## 前置条件

- Mac 和 Android 手机连接**同一个 WiFi**
- Mac 上已安装 [Homebrew](https://brew.sh)
- Android 手机可以设置 WiFi 静态 IP

---

## 步骤

### 1. 安装依赖

```bash
brew install nginx dnsmasq
```

### 2. 查看 Mac 的局域网 IP

```bash
ifconfig | grep "inet " | grep -v 127.0.0.1
```

记下形如 `192.168.x.x` 的地址，后续用 `<MAC_IP>` 表示。

### 3. 生成自签名证书

```bash
mkdir -p ~/mitm-certs
openssl req -x509 -newkey rsa:2048 \
  -keyout ~/mitm-certs/key.pem \
  -out ~/mitm-certs/cert.pem \
  -days 365 -nodes \
  -subj "/CN=gtimg.cn" \
  -addext "subjectAltName=DNS:showv6.gtimg.cn,DNS:iv6.gtimg.cn,DNS:gxh.material.qq.com,DNS:qzonestyle.gtimg.cn"
```

### 4. 配置 Nginx

将以下内容写入 `/opt/homebrew/etc/nginx/servers/qqplay.conf`（Intel Mac 路径为 `/usr/local/etc/nginx/servers/qqplay.conf`）：

```nginx
proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=qq_cache:10m max_size=100m inactive=30d use_temp_path=off;

server {
    listen 80 default_server;
    listen 443 ssl default_server;
    server_name _;

    ssl_certificate /Users/<你的用户名>/mitm-certs/cert.pem;
    ssl_certificate_key /Users/<你的用户名>/mitm-certs/key.pem;

    root /opt/homebrew/var/www;
    location / {
        try_files $uri =404;
    }
}

server {
    listen 80;
    listen 443 ssl;
    server_name showv6.gtimg.cn iv6.gtimg.cn gxh.material.qq.com qzonestyle.gtimg.cn;

    ssl_certificate /Users/<你的用户名>/mitm-certs/cert.pem;
    ssl_certificate_key /Users/<你的用户名>/mitm-certs/key.pem;

    resolver 8.8.8.8 valid=30s;

    location / {
        proxy_pass https://$host$request_uri;
        proxy_ssl_server_name on;
        proxy_set_header Host $host;
        proxy_cache qq_cache;
        proxy_cache_valid 200 30d;
        proxy_cache_use_stale error timeout;
    }
}
```

> 将 `<你的用户名>` 替换为实际用户名（通过 `whoami` 查看）。

把证书复制到 Nginx 静态目录供手机下载：

```bash
cp ~/mitm-certs/cert.pem /opt/homebrew/var/www/cert.pem
```

启动 Nginx（需要监听 80/443 端口，须用 sudo）：

```bash
sudo /opt/homebrew/bin/nginx
```

### 5. 配置 dnsmasq

```bash
cat >> /opt/homebrew/etc/dnsmasq.conf << 'EOF'
server=8.8.8.8
address=/showv6.gtimg.cn/<MAC_IP>
address=/iv6.gtimg.cn/<MAC_IP>
address=/gxh.material.qq.com/<MAC_IP>
address=/qzonestyle.gtimg.cn/<MAC_IP>
EOF
```

> 将 `<MAC_IP>` 替换为第 2 步查到的地址。

启动 dnsmasq：

```bash
sudo /opt/homebrew/sbin/dnsmasq --listen-address=<MAC_IP> --bind-interfaces
```

验证 DNS 劫持是否生效：

```bash
dig @<MAC_IP> showv6.gtimg.cn
```

ANSWER SECTION 应返回 `<MAC_IP>`，说明配置正确。

### 6. 手机安装证书

手机浏览器访问：

```
http://<MAC_IP>/cert.pem
```

下载后安装：**设置 → 安全 → 加密与凭据 → 安装证书 → CA 证书**，选择下载的文件。

### 7. 手机设置静态 IP 和 DNS

进入 WiFi 详情，将 IP 设置改为**静态**：

| 字段 | 值 |
|------|-----|
| IP 地址 | 原来的 IP（如 `192.168.2.100`） |
| 网关 | 路由器 IP（如 `192.168.2.1`） |
| 子网掩码前缀 | 24 |
| DNS 1 | `<MAC_IP>` |
| DNS 2 | `8.8.8.8` |

保存后断开 WiFi 重新连接。

### 8. 切换夜间模式

打开 QQPLAY，进入主题设置，切换夜间模式，等待下载完成即可。

---

## 使用完毕后关闭

```bash
sudo /opt/homebrew/bin/nginx -s stop
sudo kill $(pgrep dnsmasq)
```

手机 WiFi 的 IP 设置改回 **DHCP**。

---

## 常见问题

**手机连不上网**  
检查静态 IP 的前三段是否和网关一致，例如网关是 `192.168.2.1`，手机 IP 必须是 `192.168.2.x`。

**主题依然下载失败**  
在 Mac 终端运行以下命令查看实时日志，同时在手机上操作，看是否有请求进来：
```bash
sudo tail -f /opt/homebrew/var/log/nginx/access.log
```

**证书安装后仍报不信任**  
部分 Android 系统需要在安装后手动启用：**设置 → 安全 → 加密与凭据 → 信任的凭据 → 用户**，确认证书已勾选。

**Mac 重启后需要重新运行**  
每次重启后需要重新执行第 4 步（启动 Nginx）和第 5 步（启动 dnsmasq）的启动命令。

---

## 注意事项

- 切换主题完成后建议立即关闭服务，避免长期影响手机网络
```
sudo /opt/homebrew/bin/nginx -s stop
sudo kill $(pgrep dnsmasq)
```
- 本方案不收集任何数据，所有流量仅在局域网内转发
- Nginx 启用了缓存（30天），同一主题包再次下载时直接从本地返回，无需重新联网
