# VPS Proxy Exit Node Skill

[English README](README.md)

`vps-proxy-exitnode` 是一个 Codex skill，用来把一台 Linux VPS 配置成“Tailscale Exit Node + 3X-UI/Xray VLESS Reality 代理节点”。

它沉淀了一套可重复执行的服务器部署流程：当你提供服务器 IP、SSH 用户名和密码后，Codex 可以按照这套流程完成 SSH 初始化、Tailscale Exit Node 配置、3X-UI/Xray 部署、Clash/Mihomo 订阅生成，以及 HTTPS 后台和订阅托管。

## 能做什么

- 将密码 SSH 登录转换为本机密钥登录。
- 安装或检查 Tailscale，并将 VPS 广播为 Exit Node。
- 开启并持久化 IPv4/IPv6 转发。
- 部署 3X-UI/Xray，并创建 VLESS Reality 入站。
- 通过 `proxy-fleet` 生成和同步 Clash/Mihomo 订阅。
- 使用 Nginx 和 Let’s Encrypt 配置 HTTPS 后台与订阅访问。
- 支持隐藏 3X-UI 后台路径，并让面板只监听本机。
- 支持与同机已有 DERP 服务共存。
- 提供 Tailscale、Nginx、3X-UI、Xray、订阅链接的健康检查流程。

## 适用场景

当你希望 Codex 配置或检查以下服务时，可以使用这个 skill：

- Tailscale Exit Node
- 3X-UI / Xray 代理服务
- VLESS Reality 节点
- Clash / Mihomo 订阅托管
- 通过域名访问 HTTPS 后台
- DERP 与 Nginx 共用 `80` / `443` 端口

## 仓库结构

```text
.
├── SKILL.md                 # skill 触发信息和高层工作流
├── references/
│   └── runbook.md           # 详细部署与排错手册
├── agents/
│   └── openai.yaml          # Codex UI 元信息
├── README.md                # 英文简介
├── README.zh-CN.md          # 中文简介
└── LICENSE
```

## 安装方式

将这个 skill 克隆到 Codex 的 skills 目录：

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/yaoleifly/vps-proxy-exitnode-skill.git ~/.codex/skills/vps-proxy-exitnode
```

之后就可以让 Codex 按这个流程配置服务器。

示例请求：

```text
我的服务器 IP 是 <ip>，用户名是 <user>，密码是 <password>。
请使用 VPS Proxy Exit Node skill，把它配置为 Tailscale Exit Node 和 3X-UI 代理节点。
```

## 需要提供的信息

最低需要：

- VPS IP 地址
- SSH 用户名
- SSH 密码

可选但推荐：

- 用于 HTTPS 的域名或子域名
- Tailscale auth key，或者由你手动完成 Tailscale 登录链接
- 节点显示名称和地区 emoji

## 安全说明

本仓库不会包含真实服务器密码、私钥、面板密码、GitHub token、Tailscale key 或任何机器专属密钥。

实际使用时建议：

- 将服务器登录信息视为临时敏感信息。
- 使用后及时撤销或轮换 token。
- 不要让 3X-UI 后台裸露在明文 HTTP 上。
- 优先让 3X-UI 面板只监听 `127.0.0.1`，再通过 Nginx HTTPS 暴露。
- 除非明确知道影响，否则 Cloudflare 对代理类流量应保持“仅 DNS”。

## 许可证

MIT
