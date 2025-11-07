---
title: '在 NAT VPS 上部署哪吒监控完整指南'
pubDate: '2025-11-07'
---

## 背景说明

NAT VPS 没有公网 IP，需要通过端口映射或反向隧道实现外网访问。

------

## 第一步：安装哪吒监控面板

### 1.1 下载并运行安装脚本

bash

```
curl -L https://raw.githubusercontent.com/nezhahq/scripts/refs/heads/main/install.sh -o nezha.sh && chmod +x nezha.sh && sudo ./nezha.sh
```



### 1.2 安装配置选项

- **面板运行端口**：选择 `8008`（或其他可用端口）
- **访问地址**：输入 `127.0.0.1:8008`
- **启用 TLS**：选择 `N`

### 1.3 NAT 端口映射

在 VPS 控制面板设置端口映射：

- 外部端口：`52777`（或其他厂商提供的端口）
- 内部端口：`8008`
- 映射关系：`52777 → 8008`

------

## 第二步：配置 Cloudflare 隧道

### 2.1 登录 Cloudflare

bash

```
cloudflared tunnel login
```



- 访问生成的 URL 完成认证
- 证书文件会自动下载到 `/root/.cloudflared/`

### 2.2 创建隧道

bash

```
cloudflared tunnel create nezha
```



隧道配置文件会保存在 `/root/.cloudflared/xxx-xxx-xxx-xx.json`

### 2.3 绑定 DNS 记录

bash

```
cloudflared tunnel route dns nezha nezha.yourdomain.com
```



将生成 CNAME 记录指向 Cloudflare Tunnel

------

## 第三步：配置并运行隧道服务

### 3.1 创建配置文件

编辑 `/root/.cloudflared/config.yml`：

yaml

```
tunnel: xxx-xxx-xxx-xx
credentials-file: /root/.cloudflared/xxx-xxx-xxx-xx.json

ingress:
  - hostname: nezha.yourdomain.com
    service: http://localhost:8008
  - service: http_status:404
```



### 3.2 安装系统服务

bash

```
cloudflared service install
systemctl enable cloudflared
systemctl start cloudflared
```



### 3.3 验证服务状态

bash

```
systemctl status cloudflared
```



现在可以通过 `https://nezha.yourdomain.com` 访问哪吒面板。

------

## 第四步：添加监控客户端（探针）

### 4.1 普通 VPS（有公网 IP）

bash

```
curl -L https://raw.githubusercontent.com/nezhahq/scripts/main/agent/install.sh -o agent.sh && chmod +x agent.sh && env NZ_SERVER=23.325.36.5:52777 NZ_TLS=false NZ_CLIENT_SECRET=uq7vHTMgbgXkFQSfBPgmMASIPLU2bDgO ./agent.sh
```



**参数说明：**

- `NZ_SERVER`: VPS 公网 IP + 映射端口
- `NZ_CLIENT_SECRET`: 在面板中生成的客户端密钥

### 4.2 NAT VPS 本机添加

bash

```
curl -L https://raw.githubusercontent.com/nezhahq/scripts/main/agent/install.sh -o agent.sh && chmod +x agent.sh && env NZ_SERVER=127.0.0.1:8008 NZ_TLS=false NZ_CLIENT_SECRET=viFBd9wsJq8qKbONRbS7XMtXEBy7kUUD ./agent.sh
```



**参数说明：**

- `NZ_SERVER`: 本地地址 + 面板端口
- `NZ_CLIENT_SECRET`: 在面板中生成的客户端密钥

------

## 第五步：验证和管理

### 5.1 检查探针状态

bash

```
systemctl status nezha-agent
journalctl -u nezha-agent -f
```



### 5.2 管理命令

bash

```
# 重启探针服务
systemctl restart nezha-agent

# 查看隧道状态
cloudflared tunnel list

# 检查服务日志
journalctl -u cloudflared -f
```



------

## 注意事项

1. **域名准备**：确保域名已托管到 Cloudflare
2. **端口开放**：确认 VPS 厂商的防火墙规则允许相应端口
3. **密钥安全**：妥善保管客户端密钥，不要泄露
4. **HTTPS 自动配置**：Cloudflare 会自动为域名配置 HTTPS 证书

通过以上步骤，即可在 NAT VPS 上成功部署哪吒监控系统，并通过 Cloudflare 隧道实现外网安全访问。
