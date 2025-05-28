

ZeroTier 是一种强大的虚拟网络工具，可以创建虚拟局域网（VLAN），让远程设备像在同一个本地网络中一样通信。结合OpenWrt（一个功能丰富的开源路由器固件），ZeroTier可以实现远程访问本地网络设备、构建安全VPN等场景。本文将指导您在OpenWrt虚拟机上安装和配置ZeroTier，连接到物理LAN（例如192.168.31.0/24），并解决常见的配置问题。

## 为什么选择ZeroTier和OpenWrt？

ZeroTier通过点对点的加密网络连接设备，无需复杂端口映射，适合远程访问家庭或办公室网络。OpenWrt则以其灵活的网络配置和强大的防火墙功能，成为运行ZeroTier的理想平台。通过在OpenWrt上配置ZeroTier，您可以：

- 让外部设备通过ZeroTier访问OpenWrt路由器（SSH或Web界面）。
- 访问同一物理LAN中的其他设备。
- 实现ZeroTier网络与物理LAN之间的双向通信。

本文假设您已有一个ZeroTier网络ID，OpenWrt虚拟机运行在VMware或VirtualBox中，连接到物理LAN（192.168.31.0/24）。以下是详细步骤和故障排除指南。

## 安装和配置ZeroTier的步骤

### 1. 安装ZeroTier

首先，通过SSH或LuCI（OpenWrt的Web界面）连接到您的OpenWrt虚拟机，然后执行以下命令安装ZeroTier：

```bash
opkg update
opkg install zerotier
/etc/init.d/zerotier enable
/etc/init.d/zerotier start
```

这会更新软件包列表，安装ZeroTier，并启用开机自启。

### 2. 加入ZeroTier网络

1. 使用以下命令加入您的ZeroTier网络，替换`<YOUR_NETWORK_ID>`为实际的网络ID：

   ```bash
   zerotier-cli join <YOUR_NETWORK_ID>
   ```

   成功后，您将看到类似`200 join OK`的输出。

2. 登录`my.zerotier.com`，在您的网络“Members”部分找到OpenWrt设备（通过`zerotier-cli info`获取其地址），勾选“Auth?”授权，并记录分配的ZeroTier IP（例如10.x.x.x）。

3. 在ZeroTier Central的“Managed Routes”中添加路由，允许访问物理LAN：

   - **目标网络**：`192.168.31.0/24`
   - **下一跳 (Via)**：OpenWrt的ZeroTier IP
   - 点击“Submit”保存。

### 3. 配置OpenWrt网络接口

ZeroTier会自动创建一个接口（例如`ztabcdef12`，可用`ip addr`查看）。通过LuCI配置：

1. 导航到**网络 &gt; 接口**，点击**添加新接口**。
2. 名称设为`ZeroTier_VPN`，协议选择**不配置协议 (Unmanaged)**。
3. 选择ZeroTier接口（以`zt`开头）。
4. 在**防火墙设置**中，创建新区域`zt_vpn`。
5. 保存并应用。

### 4. 配置防火墙规则

防火墙配置是确保ZeroTier客户端访问OpenWrt和LAN设备的关键：

1. 导航到**网络 &gt; 防火墙**，编辑`zt_vpn`区域：
   - **输入**：`ACCEPT`（允许访问OpenWrt）。
   - **输出**：`ACCEPT`。
   - **转发**：`REJECT`（除非需要）。
   - **涵盖网络**：选择`ZeroTier_VPN`。
2. 在**区域间转发**中：
   - 允许`zt_vpn`转发到`lan`。
   - （可选）允许`lan`转发到`zt_vpn`以实现双向通信。
3. 在`lan`区域启用**IP动态伪装 (Masquerading)**，确保ZeroTier流量进入LAN时源IP转换为OpenWrt的LAN IP。
4. 保存并应用。

**提示**：如果LuCI配置复杂，可在**自定义规则**中添加：

```bash
iptables -t nat -A POSTROUTING -s <ZeroTier_Network_CIDR> -o br-lan -j MASQUERADE
```

替换`<ZeroTier_Network_CIDR>`为您的ZeroTier网段（如10.147.0.0/16）。

### 5. 验证配置

1. 检查ZeroTier状态：

   ```bash
   zerotier-cli listnetworks
   ip addr show zt<NetworkID_part>
   ```

   确认网络ID和IP分配正确。

2. 从ZeroTier客户端测试：

   - `ping <OpenWrt_ZeroTier_IP>`
   - `ssh root@<OpenWrt_ZeroTier_IP>`或访问`http://<OpenWrt_ZeroTier_IP>`
   - `ping <OpenWrt_LAN_IP>`（例如192.168.31.9）
   - `ping <LAN_Device_IP>`（例如192.168.31.100）

3. 对于LAN到ZeroTier的双向通信，若OpenWrt不是默认网关，在主路由器（例如192.168.31.1）上添加静态路由：

   - **目标网络**：ZeroTier网段（例如10.147.0.0/16）
   - **网关**：OpenWrt的LAN IP（例如192.168.31.9）

## 常见问题与故障排除

### 问题：无法ping通OpenWrt的LAN IP

如果您可以ping通OpenWrt的ZeroTier IP（例如10.x.x.x），但无法ping通其LAN IP（例如192.168.31.9），可能的原因包括：

1. **ZeroTier客户端路由问题**：

   - 确保客户端启用“允许托管路由”。

   - 检查客户端路由表（例如`ip route show`或`route print -4`），确认有指向`192.168.31.0/24`的路由，通过OpenWrt的ZeroTier IP。

   - 临时测试：在客户端手动添加路由：

     ```bash
     route add 192.168.31.0 mask 255.255.255.0 <OpenWrt_ZeroTier_IP>
     ```

2. **ZeroTier Central托管路由配置错误**：

   - 登录`my.zerotier.com`，确认“Managed Routes”中目标为`192.168.31.0/24`，下一跳是OpenWrt的ZeroTier IP。

3. **OpenWrt防火墙阻止请求**：

   - 确保`zt_vpn`区域的**输入**设置为`ACCEPT`。

   - 使用`tcpdump`检查ZeroTier接口（例如`zt7nnop44b`）是否收到ping请求：

     ```bash
     tcpdump -i zt7nnop44b -n icmp and dst host 192.168.31.9 and src host <ZT_Client_IP>
     ```

     若无数据包，问题出在客户端或ZeroTier Central；若有请求但无回复，检查防火墙日志：

     ```bash
     logread -f | grep DROP
     ```

**注意事项**：

- 确保虚拟机网络适配器为**桥接模式**，与物理LAN同网段。
- 检查ZeroTier接口名称（`zt...`）是否正确，可能因网络ID变化。
- 配置更改后，重启ZeroTier（`/etc/init.d/zerotier restart`）或网络服务。

## 总结与后续步骤

通过以上步骤，您可以在OpenWrt上成功配置ZeroTier，实现远程访问路由器和LAN设备。ZeroTier的点对点加密和OpenWrt的灵活性为家庭或小型网络提供了强大解决方案。

**后续探索**：

- 研究ZeroTier的Flow Rules以增强网络安全。
- 配置更精细的防火墙规则，限制特定流量。
- 探索ZeroTier的高级功能，如SD-WAN或多网络桥接。

有关更多信息，请访问ZeroTier官网或OpenWrt文档。