下面是一份从 0 开始、每一步都带验证的完整文档。目标：搭建一台**安全、稳定、可用于合法学习与研究**的 VPS 代理环境。主线使用 **Sing-box + VLESS Reality**，并包含 **Codex 稳定性优化**与**住宅 IP 中转**说明。

> ⚠️ 合规提醒：请确保所有操作符合当地法律法规，仅用于合法学习、研究与合规网络访问。不要用于任何非法用途。

---

# 从 0 搭建安全稳定 VPS 完整教程

## 0. 方案选型与目标

### 0.1 协议优先级

| 优先级 | 方案                   | 定位                                   |
| ------ | ---------------------- | -------------------------------------- |
| P0     | Sing-box 多协议栈      | 统一管理 VLESS Reality / Hysteria 2 等 |
| P0     | VLESS + REALITY        | 主力 TCP 节点，无需域名，抗封锁较好    |
| P1     | Hysteria 2             | 高速备用，QUIC/UDP，适合高丢包         |
| P2     | VLESS + WS + TLS + CDN | 兜底，需要域名                         |
| P3     | Trojan / Shadowsocks   | 存量兼容                               |

### 0.2 Codex 稳定性选择

为了稳定使用 Codex，出口 IP 优先级：

**静态住宅 IP > 自建 VPS 固定出口 > 机场共享代理**

- 静态住宅 IP：信誉最高，最稳，成本高。
- 自建 VPS 固定出口：可控、便宜，但仍是机房 IP。
- 机场共享代理：IP 多人共用，容易被风控，不推荐用于 Codex。

✅ 验证：明确你的目标。如果只是学习搭建，VPS + VLESS Reality 足够；如果要稳定 Codex，后续加住宅 IP 中转。

---

## 1. 选购 VPS 与首次连接

### 1.1 选购建议

| 配置 | 推荐                                |
| ---- | ----------------------------------- |
| CPU  | 1-2 核                              |
| 内存 | 1-2GB                               |
| 存储 | 20-40GB SSD                         |
| 带宽 | 100Mbps 以上                        |
| 机房 | 洛杉矶 / 日本 / 新加坡              |
| 系统 | Ubuntu 22.04/24.04 LTS 或 Debian 12 |

### 1.2 首次 SSH 连接

```bash
ssh root@你的服务器IP
```

✅ 验证：终端提示符变为 `root@主机名:~#`，表示登录成功。

---

## 2. 系统初始化与安全加固

### 2.1 更新系统

```bash
apt update && apt upgrade -y
```

✅ 验证：命令无报错，末尾显示 `0 upgraded, 0 newly installed` 或类似提示。

### 2.2 创建普通用户并配置 SSH 密钥

**本地电脑生成密钥**：

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

一路回车，默认保存到 `~/.ssh/id_ed25519`。

**上传公钥到 VPS**：

```bash
ssh-copy-id root@你的服务器IP
```

**在 VPS 上创建新用户**：

```bash
adduser yourname
usermod -aG sudo yourname
```

**把 root 的 authorized_keys 复制给新用户**：

```bash
mkdir -p /home/yourname/.ssh
cp /root/.ssh/authorized_keys /home/yourname/.ssh/authorized_keys
chown -R yourname:yourname /home/yourname/.ssh
chmod 700 /home/yourname/.ssh
chmod 600 /home/yourname/.ssh/authorized_keys
```

✅ 验证：执行 `su - yourname`，提示符变为 `yourname@主机名:~$`。

### 2.3 SSH 安全加固

编辑 SSH 配置：

```bash
sudo nano /etc/ssh/sshd_config
```

修改以下项：

```text
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

重启 SSH：

```bash
sudo systemctl restart ssh || sudo systemctl restart sshd
```

⚠️ 不要关闭当前窗口。新开一个终端测试：

```bash
ssh -p 2222 yourname@你的服务器IP
```

✅ 验证：新终端能成功登录，说明配置正确。确认后再关闭旧窗口。

### 2.4 配置防火墙 UFW

先放行必要端口，再启用：

```bash
sudo ufw allow 2222/tcp
sudo ufw allow 443/tcp
sudo ufw allow 8443/udp
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

如果你还没改 SSH 端口，先临时放行 22：

```bash
sudo ufw allow 22/tcp
```

改完并测试成功后，再删除 22：

```bash
sudo ufw delete allow 22/tcp
```

✅ 验证：

```bash
sudo ufw status verbose
```

应看到 `2222/tcp ALLOW`、`443/tcp ALLOW`、`8443/udp ALLOW`，状态为 `active`。

### 2.5 安装 Fail2Ban

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

配置 SSH 端口：

```bash
sudo tee /etc/fail2ban/jail.local > /dev/null <<'EOF'
[sshd]
enabled = true
port = 2222
maxretry = 5
bantime = 3600
EOF
sudo systemctl restart fail2ban
```

✅ 验证：

```bash
sudo fail2ban-client status sshd
```

应显示 `Currently banned`、`Total banned` 等信息。

### 2.6 开启 BBR 加速

```bash
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

✅ 验证：

```bash
sysctl net.ipv4.tcp_congestion_control
```

应返回：

```text
net.ipv4.tcp_congestion_control = bbr
```

---

## 3. 安装并配置 Sing-box

### 3.1 安装 Sing-box

```bash
bash <(curl -fsSL https://sing-box.app/install.sh)
```

✅ 验证：

```bash
sing-box version
```

应输出版本号。

### 3.2 生成密钥与 UUID

**生成 UUID**：

```bash
sing-box generate uuid
```

记下输出，例如：`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`。

**生成 REALITY 密钥对**：

```bash
sing-box generate reality-keypair
```

输出示例：

```text
PrivateKey: 你的私钥
PublicKey: 你的公钥
```

服务端配置用 `PrivateKey`，客户端用 `PublicKey`。

**生成 Short ID**：

```bash
sing-box generate rand --hex 8
```

记下输出，例如：`a1b2c3d4e5f60708`。

✅ 验证：三个值都保存好，不要丢失。

### 3.3 编写服务端配置

创建 `/etc/sing-box/config.json`：

```bash
sudo nano /etc/sing-box/config.json
```

写入：

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "inbounds": [
    {
      "type": "vless",
      "tag": "vless-in",
      "listen": "::",
      "listen_port": 443,
      "users": [
        {
          "uuid": "你的UUID",
          "flow": "xtls-rprx-vision"
        }
      ],
      "tls": {
        "enabled": true,
        "server_name": "www.apple.com",
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "www.apple.com",
            "server_port": 443
          },
          "private_key": "你的REALITY私钥",
          "short_id": ["你的ShortID"]
        }
      }
    }
  ],
  "outbounds": [
    {
      "type": "direct",
      "tag": "direct"
    }
  ]
}
```

> 说明：`server_name` 与 `handshake.server` 是伪装目标域名，可换成 `addons.mozilla.org` 等。`private_key` 填服务端私钥。

✅ 验证配置语法：

```bash
sing-box check -c /etc/sing-box/config.json
```

无输出或输出 `configuration OK` 即正确。

### 3.4 启动并设置开机自启

```bash
sudo systemctl enable --now sing-box
```

✅ 验证服务状态：

```bash
sudo systemctl status sing-box
```

应显示 `active (running)`。

✅ 验证日志：

```bash
sudo journalctl -u sing-box -n 50 --no-pager
```

最近 50 行无持续报错即可。

---

## 4. 客户端配置与连接验证

### 4.1 客户端选择

| 平台    | 推荐客户端                       |
| ------- | -------------------------------- |
| Windows | Clash Verge Rev、v2rayN、Hiddify |
| macOS   | Clash Verge Rev、Nekoray         |
| Android | Karing、v2rayNG                  |
| iOS     | Shadowrocket、Streisand          |
| Linux   | Clash Verge Rev、Nekoray         |

### 4.2 手动填写节点

| 参数     | 值                        |
| -------- | ------------------------- |
| 地址     | 你的 VPS IP               |
| 端口     | 443                       |
| UUID     | 第 3.2 步生成             |
| 流控     | xtls-rprx-vision          |
| 传输协议 | tcp                       |
| 安全     | reality                   |
| SNI      | www.apple.com             |
| 公钥     | 第 3.2 步生成的 PublicKey |
| Short ID | 第 3.2 步生成的 Short ID  |
| 指纹     | chrome                    |

### 4.3 连接验证

**验证 1：出口 IP**

访问 `https://ip.sb` 或 `https://api.ipify.org`，应显示你的 VPS IP。

**验证 2：DNS 泄漏**

访问 `https://dnsleaktest.com`，运行标准测试，应只显示 VPS 所在地区 DNS。

**验证 3：命令行连通性**

```bash
curl -x socks5://127.0.0.1:10808 https://www.google.com -I
```

端口按客户端本地 SOCKS 端口调整，应返回 `HTTP/2 200`。

✅ 验证：三项都通过，说明代理已可用。

---

## 5. Codex 稳定性优化

### 5.1 为什么 VPS 直连可能不够稳

VPS 的 IP 是机房 IP，信誉分低于住宅 IP。Codex 风控严格时，可能触发验证码或限制。

### 5.2 住宅 IP 中转方案

如果你有住宅 SOCKS5 代理，可以在 Sing-box 中增加出站和路由。

在 `outbounds` 中增加：

```json
{
  "type": "socks",
  "tag": "residential",
  "server": "住宅IP地址",
  "server_port": 1080,
  "version": "5",
  "username": "用户名",
  "password": "密码"
}
```

在配置中增加 `route`：

```json
"route": {
  "rules": [
    {
      "domain_suffix": [
        "openai.com",
        "chatgpt.com",
        "oaistatic.com",
        "oaiusercontent.com"
      ],
      "outbound": "residential"
    }
  ]
}
```

然后重启：

```bash
sudo systemctl restart sing-box
```

✅ 验证：

```bash
sudo systemctl status sing-box
```

应 `active (running)`。

连接后访问 `https://ip.sb`，应显示住宅 IP 的运营商信息，而非 VPS 机房 IP。

### 5.3 方案优先级总结

1. 静态住宅 IP：最稳，成本高。
2. 自建 VPS 固定出口：可控，性价比高。
3. 机场共享代理：不推荐用于 Codex。

---

## 6. 运维与常见问题排查

| 问题              | 可能原因           | 解决方法                                            |
| ----------------- | ------------------ | --------------------------------------------------- |
| SSH 连接超时      | 防火墙未放行新端口 | 通过 VPS 网页控制台登录，检查 `ufw status`          |
| Sing-box 启动失败 | 配置错误           | `sing-box check -c /etc/sing-box/config.json`       |
| 客户端连不上      | 端口未放行         | 检查 VPS 安全组和 UFW 是否放行 443                  |
| REALITY 握手失败  | SNI 域名被屏蔽     | 换 `addons.mozilla.org` 等单来源域名                |
| 速度慢            | 未开 BBR           | `sysctl net.ipv4.tcp_congestion_control` 确认 `bbr` |
| DNS 泄漏          | 客户端未走代理 DNS | 客户端开启远程 DNS 或代理 DNS                       |

常用命令：

```bash
sudo systemctl status sing-box
sudo journalctl -u sing-box -n 100 --no-pager
sing-box check -c /etc/sing-box/config.json
sudo systemctl restart sing-box
```

✅ 验证：每次修改配置后，先 `sing-box check`，再重启，再看状态。

---

## 7. 安全清单

- [ ] 已创建普通用户并禁用 root 远程登录
- [ ] 已配置 SSH 密钥认证，禁用密码登录
- [ ] 已修改 SSH 默认端口
- [ ] 已启用 UFW，仅放行必要端口
- [ ] 已安装并配置 Fail2Ban
- [ ] 已开启 BBR
- [ ] 已审阅安装脚本内容
- [ ] 已配置 DNS 防泄漏
- [ ] 已定期更新系统：`apt update && apt upgrade -y`
- [ ] 已备份 `/etc/sing-box/config.json`

---

## 8. 合规提醒

请确保所有操作符合当地法律法规，仅用于合法学习、研究与合规网络访问。不要用于任何非法用途。搭建代理服务可能受到当地法律限制，请自行确认并承担相应责任。

---

## 附录：备选 Xray 一键脚本

如果你更熟悉 Xray，可以用官方安装脚本：

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

✅ 验证：

```bash
xray version
systemctl status xray
```

注意：该命令只安装 Xray，不会自动生成可用配置。你仍需自行编写 `/usr/local/etc/xray/config.json`，并放行端口、启动服务。

---

以上就是从 0 到可用、安全、稳定的 VPS 完整教程。建议按顺序执行，每完成一步都做验证，遇到问题先查日志，再回退配置。