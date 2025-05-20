## 🧩 前提条件

1. 你已拥有：
   - 一个已注册的 ZeroTier 账户；
   - 已创建一个 ZeroTier 网络；
   - 能通过 SSH 或其他方式访问飞牛NAS；
   - 飞牛NAS运行的是 Linux 内核（通常是 Debian/Ubuntu 衍生版）。
2. 本指南假设：
   - 飞牛NAS IP（本地网段）为 `192.168.31.3`；
   - 飞牛NAS所在局域网为 `192.168.31.0/24`；
   - 你创建的 ZeroTier 网络 ID 是 `35c192ce9b753d1a`（请替换为你的实际 ID）。

------

## 🔧 步骤 1：安装 ZeroTier

### 1.1 SSH 登录飞牛NAS

```
bash


复制编辑
ssh root@192.168.31.3
```

### 1.2 安装 ZeroTier

```
bash


复制编辑
curl -s https://install.zerotier.com | bash
```

------

## 🌐 步骤 2：加入你的 ZeroTier 网络

```
bash


复制编辑
zerotier-cli join 35c192ce9b753d1a
```

然后登录 https://my.zerotier.com/，授权此设备。

------

## ⚙️ 步骤 3：配置增强版 NAT 转发脚本

### 3.1 创建脚本文件

```
bash


复制编辑
sudo nano /usr/local/bin/zt-nat.sh
```

粘贴以下增强版内容（支持日志、状态检查、安全清理）：

```
bash复制编辑#!/bin/bash

LOG_FILE="/var/log/zt-nat.log"
NET_IF="enp6s18-ovs"
DATE=$(date "+%Y-%m-%d %H:%M:%S")

log() {
    echo "[$DATE] $1" | tee -a "$LOG_FILE"
}

log "=== 启动 ZeroTier NAT 配置脚本 ==="

# 检查网卡是否存在
if ! ip link show "$NET_IF" > /dev/null 2>&1; then
    log "❌ 错误：找不到网卡 $NET_IF，脚本终止。"
    exit 1
fi

# 检查网卡是否已启动
if ! ip link show "$NET_IF" | grep -q "state UP"; then
    log "⚠️ 网卡 $NET_IF 存在但未启用，请确认网络状态。"
fi

# 启用 IP 转发
log "🔧 启用 IPv4 转发..."
sysctl -w net.ipv4.ip_forward=1 >> "$LOG_FILE" 2>&1

# 清除旧的 NAT 转发规则（避免重复添加）
log "🧹 清除旧的 NAT 规则（如果存在）..."
iptables -t nat -D POSTROUTING -o "$NET_IF" -j MASQUERADE 2>/dev/null

# 添加 NAT 转发规则
log "➕ 添加 NAT 转发规则..."
iptables -t nat -A POSTROUTING -o "$NET_IF" -j MASQUERADE

log "✅ NAT 转发配置完成，ZeroTier 虚拟局域网现在可以访问本地局域网。"
log "========================================"
```

保存并退出（`Ctrl+O`，`Enter`，`Ctrl+X`）

### 3.2 赋予执行权限

```
bash


复制编辑
sudo chmod +x /usr/local/bin/zt-nat.sh
```

### 3.3 创建日志文件

```
bash
复制编辑
sudo touch /var/log/zt-nat.log
sudo chmod 644 /var/log/zt-nat.log
```

------

## 🔁 步骤 4：创建 systemd 服务（开机自动启用）

```
bash


复制编辑
sudo nano /etc/systemd/system/zt-nat.service
```

粘贴内容如下：

```
ini复制编辑[Unit]
Description=Enable NAT for ZeroTier Gateway
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/zt-nat.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

保存并退出后执行：

```
bash复制编辑sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable zt-nat.service
sudo systemctl start zt-nat.service
```

------

## 🧪 步骤 5：ZeroTier 网络配置

1. 登录 my.zerotier.com
2. 打开你的网络设置
3. 添加路由：

```
php-template


复制编辑
192.168.31.0/24 via 192.168.196.3
```

✅ 说明：

- `192.168.31.0/24` 是你的本地局域网网段；
- `via` 后面填 NAS 在 ZeroTier 网络中的虚拟 IP（例如 `10.147.x.x`）；
- 记得打勾 `Allow Managed Routes`。

------

## 🧪 步骤 6：测试访问

1. 从加入 ZeroTier 的其他设备（如笔记本）尝试访问局域网内其他设备：

```
bash


复制编辑
ping 192.168.31.8
```

1. 如果能 ping 通说明成功！否则检查：
   - 被访问设备是否启用了 ping/防火墙允许；
   - NAS NAT 是否成功（查看 `/var/log/zt-nat.log`）；
   - ZeroTier 路由是否生效。

------

## 📌 日志样例 `/var/log/zt-nat.log`

```
csharp复制编辑[2025-05-20 22:14:33] === 启动 ZeroTier NAT 配置脚本 ===
[2025-05-20 22:14:33] 🔧 启用 IPv4 转发...
[2025-05-20 22:14:33] 🧹 清除旧的 NAT 规则（如果存在）...
[2025-05-20 22:14:33] ➕ 添加 NAT 转发规则...
[2025-05-20 22:14:33] ✅ NAT 转发配置完成，ZeroTier 虚拟局域网现在可以访问本地局域网。
```

------

## 🎉 全部完成！

你现在拥有了一个：

- **具备 ZeroTier 转发能力的飞牛NAS**；
- **安全、可维护、网段无关的 NAT 网关方案**；
- **带日志与错误检测的增强脚本**。