我已经在虚拟机中安装了 OpenWrt 路由器系统。现在，我希望在这台 OpenWrt 实例上安装并配置 ZeroTier，创建一个虚拟局域网（VLAN），以实现以下目标：
**1\. ZeroTier 网络连接：** 外部设备通过 ZeroTier 加入虚拟局域网后，可以访问 OpenWrt 路由器本身（例如通过 SSH 或 Web 界面）。

**2\. LAN 访问：** 外部设备能够访问与 OpenWrt 虚拟机位于同一物理局域网（LAN 网段，192.168.31.0/24）内的其他设备。

**3\. 网络通信：** 确保 ZeroTier 网络和物理 LAN 之间的双向通信正常工作。

请提供详细的步骤和配置指南，包括但不限于以下内容：

• 在 OpenWrt 上安装 ZeroTier 的具体命令和步骤。

• 配置 ZeroTier 网络接口以加入指定的 ZeroTier 网络（假设我已有一个 ZeroTier 网络 ID）。

• 配置 OpenWrt 防火墙规则，确保 ZeroTier 客户端能够访问 OpenWrt 路由器和物理 LAN 中的设备，同时保持网络安全。

• 任何必要的路由规则（例如静态路由）以确保流量正确转发。

• 可选：验证配置是否成功的测试步骤或命令。

请假设：

• OpenWrt 虚拟机运行在类似 VMware 或 VirtualBox 的环境中，连接到一个物理 LAN 网段（例如 192.168.31.0/24）。

• 我对 OpenWrt 的命令行界面（LuCI 或 SSH）有基本了解。

• ZeroTier 网络已经创建，并有一个有效的网络 ID 可供使用。

请以清晰、按步骤组织的格式提供答案，并包括必要的命令、配置文件内容或 LuCI 配置截图说明。如果有潜在的常见问题或注意事项，也请指出。



---

详细步骤与配置指南
------------------

### 假设前提：

* OpenWrt 虚拟机（如 VMware、VirtualBox）已安装并运行，网络连接到物理 LAN 网段 `192.168.31.0/24`。
* 您拥有 OpenWrt 的 LuCI Web 界面或 SSH 访问权限。
* 您已在 ZeroTier Central 创建了一个网络，并拥有其 **网络 ID (Network ID)**。

---

### 步骤 1：在 OpenWrt 上安装 ZeroTier

1. **通过 SSH 或 LuCI -> 系统 -> 软件包 (Software) 连接到您的 OpenWrt 路由器。**
2. **更新软件包列表：**Bash
   
   ```
   opkg update
   ```
3. **安装 ZeroTier 软件包：**Bash
   
   ```
   opkg install zerotier
   ```
4. **启动 ZeroTier 服务并使其开机自启：**Bash
   
   ```
   /etc/init.d/zerotier enable
   /etc/init.d/zerotier start
   ```

---

### 步骤 2：配置 ZeroTier 网络

1. **加入您的 ZeroTier 网络：** 将 `<YOUR_NETWORK_ID>`替换为您的实际 ZeroTier 网络 ID。
   
   Bash
   
   ```
   zerotier-cli join <YOUR_NETWORK_ID>
   ```
   
   执行后，它会显示类似 `200 join OK` 的信息。
2. **在 ZeroTier Central 授权 OpenWrt 设备：**
   
   * 登录到 `my.zerotier.com`。
   * 进入您的网络。
   * 在 "Members" 部分，您会看到一个新的未知设备（其地址是在 OpenWrt 上执行 `zerotier-cli info` 可以看到的 `address`）。
   * 勾选 "Auth?" 列下的复选框以授权该设备。
   * （可选但推荐）给这个设备一个易于识别的名称（例如 `OpenWrt-VM`）。
   * 记下 ZeroTier Central 分配给 OpenWrt 设备的 IP 地址（例如 `10.x.x.x`）。
3. **在 ZeroTier Central 配置 Managed Route (托管路由)：** 这是关键步骤，允许 ZeroTier 网络中的其他设备访问您的物理 LAN (`192.168.31.0/24`)。
   
   * 在 ZeroTier Central 的网络设置页面，找到 "Managed Routes" 部分。
   * 添加一条新路由：
     * **Destination (目标网络):** `192.168.31.0/24`
     * **(Via) (下一跳):** `<OpenWrt_ZeroTier_IP>` (这是上一步中 ZeroTier Central 分配给 OpenWrt 设备的 IP 地址)
   * 点击 "Submit"。

---

### 步骤 3：配置 OpenWrt 网络接口

ZeroTier 安装后，通常会自动创建一个对应的网络接口，名称类似于 `zt0` 或 `zt<MemberID>` 或 `ztnnnnnnnnnn` (其中 n 是随机字符) 或更常见的是 `zt` 后跟你加入的网络 ID 的一部分，例如 `ztwqeiruty`。您可以通过 `ip addr` 或 `ifconfig` 命令查看。

1. **通过 LuCI 创建/配置网络接口：**
   
   * 导航到 **网络 (Network) -> 接口 (Interfaces)**。
   * 点击 **添加新接口... (Add new interface...)**。
     * **新接口的名称 (Name of the new interface):** 输入一个有意义的名称，例如 `ZeroTier_VPN` 或 `ZT_LAN`。
     * **协议 (Protocol):** 选择 **不配置协议 (Unmanaged)** 或者 **静态地址 (Static address)**。如果 ZeroTier 客户端已从 ZeroTier Central 获取 IP，选择 "Unmanaged" 通常可以。如果选择 "Static address"，则手动输入 OpenWrt 在 ZeroTier 网络中的 IP 地址（如果 ZeroTier Central 支持手动分配且您已配置）。通常，ZeroTier 会自动处理 IP，所以 "Unmanaged" 或让它自行获取即可。
     * **包括以下接口 (Cover the following interfaces):** 选择 ZeroTier 创建的实际网络设备名 (例如 `ztabcdef12`，您可以通过 `ls /sys/class/net` 或 `ip link show` 查找以 `zt` 开头的接口)。
   * 点击 **创建接口 (Create interface)**。
2. **配置接口的基本设置 (如果选择了静态地址或需要调整)：**
   
   * 如果您选择了 "静态地址"，请填写从 ZeroTier Central 获取的 IP 地址和子网掩码。
   * 通常，ZeroTier 会自动处理 IP，所以主要目的是将此接口关联到一个防火墙区域。
3. **配置接口的防火墙设置：**
   
   * 在接口编辑页面的 **防火墙设置 (Firewall Settings)** 标签下。
   * **创建 / 分配防火墙区域 (Create / Assign firewall-zone):** 在下拉菜单中选择 **\-- 自定义 -- (-- custom --)**，然后输入一个新的区域名称，例如 `zt_vpn`。按回车或点击空白处确认。
4. **保存并应用配置：** 点击页面底部的 **保存 (Save)**，然后点击 **保存并应用 (Save & Apply)**。

---

### 步骤 4：配置 OpenWrt 防火墙规则

目标是允许 ZeroTier 客户端访问 OpenWrt 本身以及物理 LAN (`192.168.31.0/24`)。

1. **导航到 网络 (Network) -> 防火墙 (Firewall)。**
2. **编辑新创建的 `zt_vpn` 区域：**
   
   * 找到名为 `zt_vpn` (或您自定义的名称) 的区域，点击 **编辑 (Edit)**。
   * **常规设置 (General Settings):**
     * **名称 (Name):** `zt_vpn`
     * **输入 (Input):** `ACCEPT` (允许从 ZeroTier 客户端访问 OpenWrt 路由器本身，例如 SSH, LuCI)。
     * **输出 (Output):** `ACCEPT` (允许 OpenWrt 路由器发起连接到 ZeroTier 客户端)。
     * **转发 (Forward):** `REJECT` 或 `DROP` (默认，除非有特定需要允许该区域主动转发到其他区域)。
     * **涵盖的网络 (Covered networks):** 确保这里列出了您在上一步创建的 ZeroTier 接口 (例如 `ZeroTier_VPN`)。
   * **区域间的转发 (Inter-Zone Forwarding):**
     * 在 **允许转发到目标区域 (Allow forward to destination zones):** 中，选择 `lan`。这样就允许从 `zt_vpn` 区域的流量转发到 `lan` 区域。
     * (可选，为实现双向通信) 在 **允许从源区域转发 (Allow forward from source zones):** 中，也选择 `lan`。这将允许 `lan` 区域的设备访问 `zt_vpn` 区域的设备（这需要 LAN 设备有到 ZeroTier 网络的路由，见步骤6的说明）。
   * **IP 动态伪装 (Masquerading):**
     * 这一步很重要，使得从 ZeroTier 访问 LAN 设备时，LAN 设备认为请求来自 OpenWrt 的 LAN IP 地址，从而能正确回应。
     * **返回到防火墙区域概览页面。**
     * 找到 `lan` 区域，点击其 **编辑 (Edit)** 按钮。
     * 勾选 **IP 动态伪装 (Masquerading)** 复选框。如果您的 `lan` 区域已经为出站到 `wan` 的流量启用了伪装，这通常是默认设置，可能无需额外操作。此设置确保了从其他区域（如 `zt_vpn`）转发到 `lan` 并从 `lan` 出去的流量（如果 `lan` 不是最终目的地，例如访问 `lan` 上的设备）会被正确 NAT。更精确的控制是，您可以在 `zt_vpn` 区域的转发设置中针对到 `lan` 的转发启用伪装，但这在 LuCI 中不直接可见，通常全局开启 `lan` 区域的伪装即可。
     * **本质上，您需要确保从 ZeroTier 网络到 LAN 网络的流量，在离开 OpenWrt 进入 LAN 时，源 IP 被转换为 OpenWrt 的 LAN IP。**
       * 如果通过 LuCI 不易精确配置，可以通过自定义规则实现。在 **网络 -> 防火墙 -> 自定义规则 (Custom Rules)** 中添加（如果需要）：Bash
         
         ```
         # 确保替换 <ZeroTier_Network_CIDR> 为您的 ZeroTier IP 地址范围, e.g., 10.147.0.0/16
         # 确保替换 <LAN_Interface_Name> 为您的 LAN 接口名, e.g., br-lan
         # iptables -t nat -A POSTROUTING -s <ZeroTier_Network_CIDR> -o <LAN_Interface_Name> -j MASQUERADE
         ```
         
         然而，通常将 `zt_vpn` 区域的流量转发到启用了 `Masquerading` 的 `lan` 区域就足够了。
3. **保存并应用防火墙配置：** 点击页面底部的 **保存并应用 (Save & Apply)**。

---

### 步骤 5：检查路由 (通常自动处理)

* **OpenWrt 路由:** OpenWrt 应该能自动处理其直接连接的网络（LAN 和新创建的 ZeroTier 接口）之间的路由。
* **ZeroTier Central Managed Route:** 这一步在步骤 2.3 中已经完成，它告诉其他 ZeroTier 客户端如何到达您的 `192.168.31.0/24` 网络。

---

### 步骤 6：验证配置

1. **检查 ZeroTier 接口状态：** 在 OpenWrt 的 SSH 中执行：
   
   Bash
   
   ```
   zerotier-cli listnetworks
   # 应该能看到您的网络 ID，状态为 OK，以及分配到的 IP 地址。
   # 例如：<NetworkID> <Name> <MAC> <Status> <Type> <Device> <IP_Addresses>
   # ... <YOUR_NETWORK_ID> your_network_name dead:beef:0123 ONLINE PRIVATE zabcdef12 10.x.x.x/24
   
   ip addr show zt<NetworkID_part> # (替换为实际接口名)
   # 检查接口是否有从 ZeroTier 网络获取的 IP 地址。
   ```
2. **从外部 ZeroTier 客户端测试：**
   
   * 确保外部设备已连接到同一个 ZeroTier 网络。
   * **Ping OpenWrt 的 ZeroTier IP:** `ping <OpenWrt_ZeroTier_IP>`
   * **SSH 或 Web 访问 OpenWrt:** `ssh root@<OpenWrt_ZeroTier_IP>` 或在浏览器中打开 `http://<OpenWrt_ZeroTier_IP>`。
   * **Ping OpenWrt 的 LAN IP (如果允许):** `ping <OpenWrt_LAN_IP>` (例如 `192.168.31.x`，OpenWrt 虚拟机在物理 LAN 上的 IP)。
   * **Ping 物理 LAN 中的其他设备:** `ping <IP_of_Device_on_192.168.31.0/24>` (例如 `ping 192.168.31.100`)。
3. **双向通信 (从 LAN 访问 ZeroTier 设备):**
   
   * 要使物理 LAN (`192.168.31.0/24`) 中的设备能够发起连接到 ZeroTier 客户端，这些 LAN 设备需要知道如何路由到 ZeroTier 网络。
   * **如果 OpenWrt 虚拟机是您 `192.168.31.0/24` 网络的默认网关**，则此步骤可能不需要额外配置。
   * **如果 OpenWrt 不是默认网关**，您需要在您的主物理路由器（例如 `192.168.31.1`）上添加一条静态路由：
     * **目标网络 (Destination Network):** `<Your_ZeroTier_Network_CIDR>` (例如 `10.147.0.0/16`，根据您 ZeroTier Central 的设置)
     * **子网掩码 (Subnet Mask):** (对应 CIDR 的掩码)
     * **网关/下一跳 (Gateway/Next Hop):** `<OpenWrt_VM_LAN_IP>` (例如 `192.168.31.x`，OpenWrt 虚拟机在物理 LAN 上的 IP 地址)
   * 同时，确保防火墙设置中 `lan` 区域允许转发到 `zt_vpn` 区域 (步骤 4.2 中已提及)。

---

### 步骤 7：常见问题与注意事项

* **防火墙问题：** 最常见的问题是防火墙配置不正确。仔细检查区域分配、转发规则和输入规则。可以使用 `logread -f` 在 OpenWrt 上查看防火墙日志，辅助排错。
* **ZeroTier Central 授权：** 确保 OpenWrt 设备已在 ZeroTier Central 中被授权，并且 Managed Route 配置正确无误。
* **接口名称：** ZeroTier 创建的接口名称 (`zt...`) 可能会变化或难以预测。使用 `ip link show` 或 LuCI 界面仔细确认。
* **MTU 问题：** 在极少数情况下，ZeroTier 的 MTU (通常是 1280 或更高，取决于底层网络) 可能与某些应用或网络路径不兼容。ZeroTier 通常会自动处理。
* **虚拟机网络模式：** 确保您的 OpenWrt 虚拟机的网络适配器设置为桥接模式 (Bridged Adapter)并连接到正确的物理网络接口，这样它才能与您的物理 LAN (`192.168.31.0/24`) 处于同一网段。
* **双向通信的复杂性：** 从 LAN 主动连接到 ZeroTier 设备通常比反向连接更复杂，因为它依赖于 LAN 侧的路由配置。如果仅需 ZeroTier 客户端访问 LAN，则 LAN 侧路由非必需（归功于 NAT/伪装）。
* **重启服务：** 更改配置后，有时需要重启 ZeroTier 服务 (`/etc/init.d/zerotier restart`) 或网络服务 (`/etc/init.d/network restart`)，或者直接重启 OpenWrt 虚拟机。

---

请按照这些步骤操作。如果在特定步骤遇到问题，请提供详细的错误信息或配置截图，以便进一步协助。

---

我刚按照步骤完成了，从外部 ZeroTier客户端 ping openwrt的 ZeroTier IP可以ping通，但无法ping通OpenWrt 的 LAN IP，这个是什么原因导致的

---

显示思路

这通常意味着 ZeroTier 网络本身是通的，但 OpenWrt 的防火墙或路由配置阻止了从 ZeroTier 网络访问其 LAN 接口IP。

以下是一些可能的原因和排查步骤：

1. **检查 OpenWrt 防火墙 `zt_vpn` 区域的 “输入 (Input)” 设置：**
   
   * 登录 LuCI。
   * 导航到 **网络 (Network) -> 防火墙 (Firewall)**。
   * 找到您为 ZeroTier 接口创建的防火墙区域（例如 `zt_vpn`）。
   * 点击 **编辑 (Edit)**。
   * 确保 **输入 (Input)** 设置为 **ACCEPT (接受)**。这条规则的目的是允许从 ZeroTier 网络访问 OpenWrt 路由器本身（包括其所有接口上的 IP 地址，如 SSH、LuCI 界面等）。
   * 如果这里是 `REJECT` 或 `DROP`，那么来自 ZeroTier 网络的流量（包括 ping）将无法到达 OpenWrt 的任何 IP 地址（包括 LAN IP）。
   * 修改后，点击 **保存并应用 (Save & Apply)**。
2. **检查是否有其他冲突的防火墙规则：**
   
   * 在 **网络 (Network) -> 防火墙 (Firewall) -> 通信规则 (Traffic Rules)** 中，检查是否有自定义规则可能阻止了来自 ZeroTier 网络 (源区域 `zt_vpn` 或源 IP 为 ZeroTier 网段) 的 ICMP (ping) 请求。
3. **使用 `tcpdump` 在 OpenWrt 上抓包诊断：** 如果您不熟悉 `tcpdump`，请谨慎操作。
   
   * 首先，您可能需要安装它。通过 SSH 连接到 OpenWrt 后运行：Bash
     
     ```
     opkg update
     opkg install tcpdump
     ```
   * 然后，在 ZeroTier 接口上监听 ICMP 包，看看 ping 请求是否到达 OpenWrt。假设您的 ZeroTier 接口名为 `ztabcdef12` (请用 `ip addr` 或 `zerotier-cli listnetworks` 确认您的实际接口名)，您的 ZeroTier 客户端 IP 为 `<ZT_CLIENT_IP>`，OpenWrt 的 LAN IP 为 `<OPENWRT_LAN_IP>` (例如 `192.168.31.1`):Bash
     
     ```
     tcpdump -i ztabcdef12 -n icmp and host <ZT_CLIENT_IP> and dst host <OPENWRT_LAN_IP>
     ```
     
     当您从 ZeroTier 客户端 ping OpenWrt 的 LAN IP 时：
     
     * **如果您在这里看不到任何数据包**，表示 ping 请求甚至没有通过 ZeroTier 网络到达 OpenWrt 的 ZeroTier 接口。这可能指向 ZeroTier Central 的路由配置问题（但您提到能 ping 通 OpenWrt 的 ZT IP，所以这条路径应该部分正常）或客户端的 ZeroTier 配置问题。
     * **如果您能看到 ICMP echo request 数据包到达**，但没有 ping 通，说明 OpenWrt 收到了请求，但没有响应或响应被阻止了。这进一步指向 OpenWrt 本地的防火墙或内部路由问题。
4. **在 OpenWrt 上启用防火墙日志记录：** 这可以帮助您看到哪些数据包被拒绝了。
   
   * 在 LuCI 中，进入 **网络 (Network) -> 防火墙 (Firewall)**。
   * 在各个区域（特别是 `zt_vpn` 和 `lan`）的编辑页面，通常在“高级设置”或类似标签下，可以找到 “记录被拒绝/忽略的数据包 (Log dropped/rejected packets)” 的选项，启用它。
   * 或者，通过 SSH 添加临时的日志规则（这更高级，如果通过 LuCI 操作不便时使用）。例如，在 `INPUT` 链的末尾（但在最终的 `DROP` 规则之前，如果适用）添加：Bash
     
     ```
     iptables -A INPUT -j LOG --log-prefix "INPUT_DROP: " --log-level 7
     ```
     
     (注意：这条规则会记录所有被 `INPUT` 链最终策略丢弃的包，可能会产生大量日志。)
   * 然后尝试 ping，并通过 SSH 查看日志：Bash
     
     ```
     logread -f
     ```
   * 查找与您的 ZeroTier 客户端 IP 和 OpenWrt LAN IP 相关的被拒绝的 ICMP 数据包。
5. **确认 OpenWrt 的 LAN IP 和接口：**
   
   * 在 OpenWrt 的 SSH 中运行 `ip addr show br-lan` (或者您 LAN 口对应的接口名，通常是 `br-lan`)。
   * 确认 `<OPENWRT_LAN_IP>` (例如 `192.168.31.x`) 正确配置在该接口上并且接口处于 `UP` 状态。

**最常见的原因是第一点：`zt_vpn` 防火墙区域的 `Input` 策略没有设置为 `ACCEPT`。**

请先从第一点开始检查。如果问题依旧，尝试 `tcpdump` 和防火墙日志来获取更多线索。

---

openwrt所在局域网的网关为192.168.31.1，openwrt是作为旁路由使用的，是否与这个有关系

---

显示思路

是的，OpenWrt 作为旁路由（Side Router）这个情况**非常关键**，并且很可能与您描述的问题（无法 ping 通 OpenWrt 的 LAN IP）有关，即使您能 ping 通其 ZeroTier IP。

当 OpenWrt 作为旁路由时，它的 LAN 口连接到主局域网，并且它自身**不是**这个局域网中其他设备的主要网关。这意味着：

* OpenWrt 的 LAN 接口（例如 `br-lan`，IP 地址为 `192.168.31.X`）有一个网关设置，指向您的主路由器（`192.168.31.1`）。
* ZeroTier 客户端发起的 ping 请求，目标是 OpenWrt 的 LAN IP (`192.168.31.X`)。

**问题分析：应答包的路由可能出错**

1. **Ping 请求到达：**
   
   * ZeroTier 客户端 (`ZT_Client_IP`) -> OpenWrt 的 ZeroTier IP (`OpenWrt_ZT_IP`) -> 经过 ZeroTier 网络 -> 到达 OpenWrt 的 ZeroTier 接口 (`zt_interface`)。
   * ZeroTier 客户端 (`ZT_Client_IP`) -> OpenWrt 的 LAN IP (`192.168.31.X`) -> 经过 ZeroTier 网络 -> 到达 OpenWrt 的 ZeroTier 接口 (`zt_interface`)。
   * 在这两种情况下，如果防火墙允许（`zt_vpn` 区域 `Input: ACCEPT`），OpenWrt 核心应该会收到这个 ICMP 请求。
2. **Ping OpenWrt 的 ZT IP (工作正常时)：**
   
   * 请求包：源 `ZT_Client_IP`，目标 `OpenWrt_ZT_IP`。
   * 应答包：源 `OpenWrt_ZT_IP`，目标 `ZT_Client_IP`。
   * OpenWrt 查看路由表，发现 `ZT_Client_IP` 是通过 `zt_interface` 直接可达的，所以应答包从 `zt_interface` 发出。这通常没问题。
3. **Ping OpenWrt 的 LAN IP (出现问题时)：**
   
   * 请求包：源 `ZT_Client_IP`，目标 `OpenWrt_LAN_IP` (`192.168.31.X`)。
   * **关键点：应答包的源 IP 地址将是 `OpenWrt_LAN_IP` (`192.168.31.X`)，目标是 `ZT_Client_IP`。**
   * 现在 OpenWrt 需要决定从哪个接口发送这个应答包。
     * 它知道 `ZT_Client_IP` 属于 ZeroTier 网络，应该通过 `zt_interface` 发送。
     * **但是，因为应答包的源 IP 是 `192.168.31.X` (其 LAN IP)，某些操作系统或路由策略可能会尝试从拥有该 IP 的接口（即 LAN 接口 `br-lan`）发送这个包。**
     * 如果应答包试图从 LAN 接口 (`br-lan`) 发出，它的下一跳将是 OpenWrt 的 LAN 口配置的网关，即您的主路由 `192.168.31.1`。
     * 您的主路由 `192.168.31.1` 并不知道如何到达 `ZT_Client_IP`（除非您在主路由上也为 ZeroTier 网络配置了静态路由指向 OpenWrt 的 LAN IP，但这通常是为了 LAN 内其他设备访问 ZT），所以应答包会丢失。

**这就是旁路由模式下常见的问题：多宿主主机（Multi-homed Host）的应答路由问题。**

**如何排查和可能的解决方向：**

1. **再次确认防火墙 `zt_vpn` 区域的 `Input` 设置为 `ACCEPT`：** 这是前提条件，确保 OpenWrt 至少会接收发往其 LAN IP 的入站 ICMP 请求。如果这里是 `DROP` 或 `REJECT`，那么连应答的机会都没有。
2. **使用 `tcpdump` 捕获应答包的路径：** 这是诊断问题的最有效方法。
   
   * SSH 登录 OpenWrt。
   * **在 OpenWrt 的 LAN 接口上监听 ICMP：** (假设 LAN 接口是 `br-lan`，OpenWrt 的 LAN IP 是 `192.168.31.X`，ZeroTier 客户端 IP 是 `<ZT_Client_IP>`)
     
     Bash
     
     ```
     tcpdump -i br-lan -n icmp and src host 192.168.31.X and dst host <ZT_Client_IP>
     ```
     
     当您从 ZeroTier 客户端 ping OpenWrt 的 `192.168.31.X` 时，如果您在这个 `tcpdump` 中看到了 ICMP echo reply，那就意味着 OpenWrt 正试图从错误的接口（LAN 口）回应，导致失败。
   * **同时（或之后），在 OpenWrt 的 ZeroTier 接口上监听 ICMP：** (假设 ZT 接口是 `ztabcdef12`)
     
     Bash
     
     ```
     tcpdump -i ztabcdef12 -n icmp and src host 192.168.31.X and dst host <ZT_Client_IP>
     ```
     
     如果您在这里看到了 ICMP echo reply，那说明路由是正确的，问题可能在其他地方（例如 ZT 客户端的防火墙，或者路径上的其他问题，但这与您能 ping 通 ZT IP 的情况矛盾）。
3. **检查 OpenWrt 的路由表：**
   
   Bash
   
   ```
   ip route show
   ip rule show
   ```
   
   * 确保有一条明确的路由指向您的 ZeroTier 网络段，通过 ZeroTier 接口出去。例如：`10.147.0.0/16 dev ztabcdef12 scope link src <OpenWrt_ZT_IP>`。
   * `ip rule show` 可以查看是否有策略路由规则影响了源地址为 LAN IP 的包的路由行为。

**可能的解决方案 (如果确认是应答路由问题)：**

* **策略路由 (Policy Routing / Source-Based Routing):** 您可以配置 OpenWrt，使其根据数据包的源 IP 地址来选择不同的路由表或路由规则。对于源 IP 是 `192.168.31.X` (OpenWrt LAN IP) 且目标是 ZeroTier 网络的流量，强制其通过 ZeroTier 接口出去。这通常需要创建一个新的路由表，并将特定流量导向该表。 这比较复杂，但通常是解决此类问题的标准方法。
  
  例如（概念性，具体命令需适配 OpenWrt）：
  
  1. 创建一个新的路由表 (例如叫 `from_lan_ip_table`)。
  2. 在这个新表中，为 ZeroTier 网络添加一条通过 `zt_interface` 的路由，并且可能还需要一条默认路由也走 `zt_interface` (如果需要此源 IP 的流量完全通过 ZT 出去)。
  3. 添加一条 `ip rule`，规定凡是源 IP 为 `192.168.31.X` 的流量，都使用 `from_lan_ip_table` 这个路由表。
  
  Bash
  
  ```
  # 示例，非直接可用，需调整
  # echo "100 from_lan_ip" >> /etc/iproute2/rt_tables
  # ip route add <ZeroTier_Network_CIDR> dev <zt_interface_name> table from_lan_ip
  # ip route add default via <OpenWrt_ZT_Gateway_IF_ANY> dev <zt_interface_name> table from_lan_ip # 可能不需要默认路由，或根据情况
  # ip rule add from <OpenWrt_LAN_IP>/32 table from_lan_ip priority 100
  ```
* **修改 `rp_filter` 设置 (谨慎)：** 路径反向过滤 (`net.ipv4.conf.<interface>.rp_filter`)。将其设置为 `0` (关闭) 或 `2` (松散模式) 有时可以“解决”这类问题，但这通常不被推荐，因为它会降低安全性。不建议作为首选方案。
* **确保 `/proc/sys/net/ipv4/conf/<zt_interface>/route_localnet` 为 1 (如果适用，通常用于NAT本身IP)：** 这个参数允许将分配给本地接口的 IP 地址（通常是回环地址之外的）用作外部地址进行路由。对于您的问题，这可能不是直接原因，但值得了解。

**总结与建议：**

1. **首要任务：** 使用 `tcpdump` 确认 ICMP echo reply 是从哪个接口尝试发出的。这是判断问题根源的关键。
2. 如果确认应答包试图从 LAN 口发出，那么您需要研究如何在 OpenWrt 上实施策略路由，以确保源自 OpenWrt LAN IP 并发往 ZeroTier 网络的回复能正确地从 ZeroTier 接口发出。

旁路由的配置确实会引入这类路由复杂性。请先进行 `tcpdump` 测试。

---

在ssh中输入tcpdump -i br-lan -n icmp and src host 192.168.31.9 and dst host 192.168.196.9并运行后，屏幕提示 tcpdump: verbose output suppressed, use -v\[v\]... for full protocol decode

listening on br-lan, link-type EN10MB (Ethernet), snapshot length 262144 bytes，但是没有其他内容，无法看到 ICMP echo reply

---

显示思路

好的，您在 `br-lan` 接口上运行 `tcpdump` 后没有看到任何匹配的 ICMP echo reply，这其实是一个有用的信息。它表明：

**OpenWrt 没有尝试从其 LAN 接口 (`br-lan`) 回复这个 ping 请求。**

这排除了之前的一个主要猜测，即 OpenWrt 因为是旁路由，错误地将针对其 LAN IP 的 ping 回复发送给了主网关 (`192.168.31.1`)。如果回复包走了错误的路径 (br-lan)，您应该能在 `tcpdump` 中看到。

既然回复包没有从 `br-lan` 发出，那么现在有以下几种可能性：

1. **OpenWrt 的防火墙阻止了最初的 ICMP Echo Request：**
   
   * 尽管 ping 请求的目标是 OpenWrt 的 LAN IP (`192.168.31.9`)，但这个请求是通过 ZeroTier 网络到达 OpenWrt 的 ZeroTier 接口的。
   * 如果 OpenWrt 的防火墙（特别是与 ZeroTier 接口关联的区域，我们之前称为 `zt_vpn`）的 `INPUT` 策略不是 `ACCEPT`，或者有其他规则明确拒绝了来自 ZeroTier 网络的 ICMP 请求，那么 OpenWrt 根本就不会处理这个请求，自然也不会产生回复。
   * **这是目前最可能的怀疑点。**
2. **OpenWrt 正在通过正确的 ZeroTier 接口回复，但回复包在途中丢失，或者 ZeroTier 客户端没有收到/处理它：**
   
   * 这种情况相对少见，因为您提到可以 ping 通 OpenWrt 的 ZeroTier IP。这意味着从 OpenWrt 的 ZeroTier IP 发出的回复能够成功到达客户端。
   * 不过，回复包的源 IP 不同（一个是 OpenWrt 的 ZT IP，另一个是 OpenWrt 的 LAN IP `192.168.31.9`），理论上客户端或网络路径中非常特殊的配置可能会区别对待它们。
3. **您在运行 `tcpdump` 时，ZeroTier 客户端 (`192.168.196.9`) 没有实际发出 ping 请求，或者目标/源 IP 地址不完全匹配：**
   
   * 请确保在 `tcpdump` 运行时，确实从 `192.168.196.9` 发起了对 `192.168.31.9` 的 ping。

**下一步的排查步骤：**

现在最关键的是要在 OpenWrt 的 **ZeroTier 接口**上进行 `tcpdump`，以观察数据包的进出情况。

1. **确认 ZeroTier 接口名称：**
   
   * 在 OpenWrt 的 SSH 中运行 `ip addr` 或 `zerotier-cli listnetworks`。找到您的 ZeroTier 网络对应的接口名称，例如 `ztabcdef12` 或类似格式。
2. **在 ZeroTier 接口上监听传入的 ICMP Echo Request：**
   
   * 假设您的 ZeroTier 接口名为 `<zt_interface_name>`，OpenWrt 的 LAN IP 为 `192.168.31.9`，ZeroTier 客户端 IP 为 `192.168.196.9`。
   * 运行以下命令，并同时从 ZeroTier 客户端 ping OpenWrt 的 LAN IP (`192.168.31.9`)：Bash
     
     ```
     tcpdump -i <zt_interface_name> -n icmp and dst host 192.168.31.9 and src host 192.168.196.9
     ```
   * **观察结果：**
     
     * **如果您在这里看不到任何数据包**：这表示 ping 请求没有到达 OpenWrt 的 ZeroTier 接口。可能的原因有：ZeroTier 客户端没有正确配置路由到 `192.168.31.0/24` 网段，或者 ZeroTier Central 的托管路由 (`Managed Routes`) 配置有问题（虽然您能 ping 通 OpenWrt 的 ZT IP，说明 ZT 网络基本通畅，但具体到 LAN IP 的路由仍需确认）。
     * **如果您能看到 ICMP echo request 数据包到达**：这很好，说明请求已成功抵达 OpenWrt。接下来的问题就是 OpenWrt 是否以及如何回复。
3. **如果上一步能看到请求到达，则在 ZeroTier 接口上监听传出的 ICMP Echo Reply：**
   
   * 保持从 ZeroTier 客户端 ping OpenWrt 的 LAN IP (`192.168.31.9`)。
   * 运行以下命令：Bash
     
     ```
     tcpdump -i <zt_interface_name> -n icmp and src host 192.168.31.9 and dst host 192.168.196.9
     ```
   * **观察结果：**
     
     * **如果您在这里看不到 ICMP echo reply 数据包** (但在上一步看到了 echo request)：这强烈表明 OpenWrt 收到了请求，但由于防火墙规则或其他内部原因，没有生成或发出回复。**请重点检查 OpenWrt 防火墙 `zt_vpn` 区域的 `Input` 是否为 `ACCEPT`，并检查是否有其他规则阻止了 ICMP。**
     * **如果您能看到 ICMP echo reply 数据包从这里发出**：这说明 OpenWrt 正确地从 ZeroTier 接口回复了。如果客户端仍然 ping 不通，问题可能出在 ZeroTier 网络中途、ZeroTier 客户端的防火墙，或者客户端不接受来自 `192.168.31.9` 这个源 IP 的回复（可能性较低但存在）。
4. **再次检查并启用防火墙日志：**
   
   * 确保 OpenWrt 防火墙与 ZeroTier 接口关联的区域（例如 `zt_vpn`）的 **输入 (Input)** 设置为 **ACCEPT (接受)**。
   * 在该区域的设置中，启用“记录被拒绝/忽略的数据包”。
   * 尝试 ping 后，查看 OpenWrt 的系统日志 (`logread -f` 或 `logread | grep DROP`)，看是否有与从 `192.168.196.9` 到 `192.168.31.9` 的 ICMP 包相关的丢弃记录。

根据您在 ZeroTier 接口上 `tcpdump` 的结果，我们可以更精确地定位问题。目前来看，OpenWrt 的防火墙是最需要关注的地方。

---

运行了tcpdump -i zt7nnop44b -n icmp and dst host 192.168.31.9 and src host 192.168.196.9命令后，屏幕显示：tcpdump: verbose output suppressed, use -v\[v\]... for full protocol decode

listening on zt7nnop44b, link-type EN10MB (Ethernet), snapshot length 262144 bytes，没有其他信息

---

显示思路

这个 `tcpdump` 的结果非常关键！

您在 OpenWrt 的 ZeroTier 接口 `zt7nnop44b` 上运行了命令 `tcpdump -i zt7nnop44b -n icmp and dst host 192.168.31.9 and src host 192.168.196.9`，屏幕除了显示 `tcpdump` 正在监听外，**没有任何其他信息**。

这表明，当您的 ZeroTier 客户端 (`192.168.196.9`) ping OpenWrt 的 LAN IP (`192.168.31.9`) 时，**ICMP Echo Request 数据包根本没有到达 OpenWrt 的 ZeroTier 接口 `zt7nnop44b`。**

这意味着问题**不**在 OpenWrt 如何处理或回复这个 ping 请求（因为 OpenWrt 压根没收到它），而是出在数据包从 ZeroTier 客户端到 OpenWrt 的 ZeroTier 接口的这个路径上，或者说，ZeroTier 客户端没有正确地将发往 `192.168.31.9` 的数据包通过 ZeroTier 网络发送给 OpenWrt。

既然您能从 ZeroTier 客户端 ping 通 OpenWrt 的 ZeroTier IP，这说明：

* 客户端和 OpenWrt 都在同一个 ZeroTier 网络中且已授权。
* 客户端和 OpenWrt 之间的基本 ZeroTier 通信是正常的。

那么，为什么发往 OpenWrt LAN IP 的包没有到达呢？原因可能如下：

1. **ZeroTier 客户端的路由问题：**
   
   * 客户端 (`192.168.196.9`) 可能没有正确设置指向 `192.168.31.0/24` 网段（或具体到 `192.168.31.9`）的路由，使其通过 ZeroTier 网络。
   * **检查客户端的 ZeroTier 设置：** 确保客户端的 ZeroTier 配置中允许了“允许托管路由” (Allow Managed Routes) 或类似选项。这是至关重要的，客户端需要从 ZeroTier Central 获取并应用这些路由。
   * **检查客户端的操作系统路由表：**
     * 在 Windows 上：以管理员身份运行 `route print -4`。
     * 在 Linux 上：运行 `ip route show` 或 `netstat -rn`。
     * 在 macOS 上：运行 `netstat -rn`。 查看是否有一条路由规则将 `192.168.31.0/24` 指向 OpenWrt 的 ZeroTier IP 地址（或者通过客户端自身的 ZeroTier 接口）。
2. **ZeroTier Central 的托管路由 (Managed Routes) 配置问题：**
   
   * 登录 `my.zerotier.com`。
   * 进入您的 ZeroTier 网络设置。
   * **仔细检查“Managed Routes”部分：**
     * 目标网络 (Destination)：应该是 `192.168.31.0/24`。
     * 下一跳 (Via)：**必须是 OpenWrt 虚拟机在 ZeroTier 网络中的 IP 地址**。请务必确认这个 IP 地址是正确的，并且是您这台 OpenWrt（LAN IP 为 `192.168.31.9`）的 ZeroTier IP。
     * 是否有任何拼写错误或配置错误？
   * ZeroTier Central 的配置更改可能需要一点时间才能同步到所有客户端。
3. **ZeroTier 客户端防火墙：**
   
   * 虽然可能性较低，但客户端自身的防火墙可能会阻止出站到 `192.168.31.0/24` 网段的 ICMP 包（即使是用户发起的）。
4. **ZeroTier 网络规则 (Flow Rules)：**
   
   * 在 ZeroTier Central 的网络设置中，有一个“Flow Rules”部分。默认情况下，规则是 `accept;`，允许所有流量。检查一下是否有自定义规则意外地阻止了从 `192.168.196.x` 到 `192.168.31.x` 的通信。

**下一步的排查重点应放在 ZeroTier 客户端 (`192.168.196.9`) 和 ZeroTier Central 的路由配置上：**

1. **在 ZeroTier 客户端 (`192.168.196.9`) 上：**
   
   * **首要任务：** 确保 ZeroTier 客户端软件的设置中已启用 “允许托管路由” (Allow Managed Routes / Accept Managed Routes) 或类似选项。不同的操作系统和 ZeroTier 客户端版本，这个选项的名称可能略有不同。
   * 检查操作系统的路由表，看是否有指向 `192.168.31.0/24` 的、通过 ZeroTier 的路由。
   * 尝试重启客户端上的 ZeroTier 服务。
2. **在 ZeroTier Central (`my.zerotier.com`) 上：**
   
   * **再次确认 Managed Route：** 目标 `192.168.31.0/24`，下一跳 (Via) 是 OpenWrt 的正确 ZeroTier IP。确保没有笔误。
   * 确保 OpenWrt 设备和客户端设备都已在该网络中正确授权。

**一个临时的诊断方法：** 如果您的 ZeroTier 客户端操作系统允许，可以尝试在客户端上手动添加一条静态路由，指向 `192.168.31.0/24`，下一跳是 OpenWrt 的 ZeroTier IP。 例如，在 Windows (管理员CMD) 中：`route add 192.168.31.0 mask 255.255.255.0 <OpenWrt_ZeroTier_IP>` 如果在添加手动路由后 ping 就能通了，那么问题几乎肯定出在 ZeroTier 客户端未能自动从 ZeroTier Central 获取或应用托管路由。

既然 `tcpdump` 在 OpenWrt 的 ZT 接口上没有看到目标为 `192.168.31.9` 的 ping 包，那么 OpenWrt 本身关于防火墙或回复路由的配置暂时不是当前最直接的问题。问题更可能出在“上游”——即 ZeroTier 客户端如何将这个特定目标 IP 的流量导入 ZeroTier 网络。

---

最终在手动在客户端上添加了一条静态路由后，解决了问题，已经可以实现客户端ping通OpenWrt所在的区域网，成功组建了虚拟局域网。现在我需要你把上面这些内容整理成一份笔记，后续我想要发布在自己的博客上，算是记录。要求：不是把上面所有步骤简单罗列，而是要又一定的逻辑性和条理性。因为检查了很多次问题，不需要把所有的问题可能放上去，只要把最后添加静态路由这个原因放进去即可。

---

显示思路

Okay, here's a blog post-style note based on your experience, focusing on the core issue and the final solution.

---

成功实现 ZeroTier 访问 OpenWrt 局域网：一个关键的客户端路由配置 📝
------------------------------------------------------------------

最近，我着手将我的 OpenWrt 路由器（作为虚拟机运行）接入 ZeroTier 虚拟局域网，目标是让外部 ZeroTier 客户端不仅能访问 OpenWrt 本身，还能访问到与 OpenWrt 处于同一物理局域网（例如 `192.168.31.0/24`）内的其他设备。虽然过程有些波折，但最终成功了，关键在于一个容易被忽视的客户端路由配置。

### 初始设置与遇到的“怪象” 🤔

我在 OpenWrt 上顺利安装并配置了 ZeroTier，OpenWrt 也成功加入了我的 ZeroTier 网络。在 ZeroTier Central（`my.zerotier.com`）中，我配置了指向 OpenWrt 物理局域网 `192.168.31.0/24` 的托管路由 (Managed Route)，下一跳 (Via) 指向了 OpenWrt 的 ZeroTier IP 地址。

然而，测试时出现了一个现象：

* ✅ 我的外部 ZeroTier 客户端可以成功 ping 通 OpenWrt 的 ZeroTier IP 地址。
* ❌ 但是，该客户端无法 ping 通 OpenWrt 的 LAN IP 地址（例如 `192.168.31.9`），自然也无法访问该 LAN 网段内的其他设备。

这表明 ZeroTier 网络本身是通的，OpenWrt 与 ZeroTier 网络也连接正常，但发往 OpenWrt 物理 LAN 的流量似乎“卡”住了。

### 诊断：流量去哪儿了？ pomocí `tcpdump` 追踪 🕵️‍♂️

经过一系列排查，包括检查 OpenWrt 的防火墙规则（确保允许从 ZeroTier 区域到 LAN 区域的转发，以及对 OpenWrt 自身的访问）后，问题依旧。

转折点出现在使用 `tcpdump` 工具在 OpenWrt 的 ZeroTier 网络接口上进行抓包。当我尝试从 ZeroTier 客户端 ping OpenWrt 的 LAN IP (`192.168.31.9`) 时，我在 OpenWrt 的 ZeroTier 接口上执行了类似这样的命令：

Bash

```
tcpdump -i <OpenWrt_ZeroTier_Interface_Name> -n icmp and dst host <OpenWrt_LAN_IP> and src host <ZeroTier_Client_IP>
```

结果发现，**竟然没有任何 ICMP Echo Request 数据包到达 OpenWrt 的 ZeroTier 接口！**

这意味着问题并非出在 OpenWrt 如何处理这些数据包（因为它根本没收到），而是 **ZeroTier 客户端没有将发往 `192.168.31.9` 的数据包正确地通过 ZeroTier 网络发送给 OpenWrt**。

### 解决方案：客户端手动添加静态路由 💡

既然问题出在客户端的路由行为上，解决方案也就明朗了：需要在 ZeroTier 客户端上手动添加一条静态路由，告诉它如何到达 OpenWrt 的物理局域网。

**操作步骤（以 Windows 和 Linux 为例）：**

假设：

* OpenWrt 的物理局域网网段是 `192.168.31.0/24`。
* OpenWrt 在 ZeroTier 网络中的 IP 地址是 `<OpenWrt_ZeroTier_IP>`。

**在 Windows 客户端上 (以管理员身份运行命令提示符或 PowerShell)：**

DOS

```
route add 192.168.31.0 mask 255.255.255.0 <OpenWrt_ZeroTier_IP>
```

如果想让路由永久生效 (重启后依然存在)，可以加上 `-p` 参数：

DOS

```
route add 192.168.31.0 mask 255.255.255.0 <OpenWrt_ZeroTier_IP> -p
```

**在 Linux 客户端上：**

Bash

```
sudo ip route add 192.168.31.0/24 via <OpenWrt_ZeroTier_IP>
```

要使此路由永久生效，需要将其添加到网络配置文件中，具体方法因发行版而异（例如修改 `/etc/network/interfaces`、`/etc/netplan/*.yaml` 或使用 NetworkManager 的配置）。

**在 macOS 客户端上：**

Bash

```
sudo route add -net 192.168.31.0/24 <OpenWrt_ZeroTier_IP>
```

添加完这条静态路由后，再次尝试从 ZeroTier 客户端 ping OpenWrt 的 LAN IP (`192.168.31.9`) 以及该局域网内的其他设备，一切都通了！🎉

### 为何需要手动添加？

理论上，ZeroTier Central 的“Managed Routes”功能应该能将这些路由信息推送给客户端。但在某些情况下，客户端可能由于各种原因（例如 ZeroTier 客户端软件自身的设置，如未启用“允许托管路由”/“Allow Managed Routes”；或者操作系统未能正确应用所有推送的路由；或是路由冲突等）没有正确应用这些托管路由。

手动添加静态路由是一种强制性的、明确的指示，确保客户端操作系统知道将目标为 `192.168.31.0/24` 的流量发送给 OpenWrt 的 ZeroTier IP，再由 OpenWrt 负责路由到其物理局域网。

### 总结 ✨

通过在 ZeroTier 客户端上手动配置指向 OpenWrt 局域网的静态路由，我成功解决了无法从 ZeroTier 访问 OpenWrt LAN 的问题。如果你也遇到类似 ZeroTier 托管路由似乎未按预期工作的情况，检查客户端的路由表并尝试手动添加静态路由，或许就能迎刃而解。这算是一次宝贵的网络排错经验！

---

现在有个新问题，在OpenWrt局域网的电脑上能不能直接访问局域网外的zerotier客户端？

---

显示思路

是的，可以让 OpenWrt 局域网内的电脑直接访问局域网外的 ZeroTier 客户端，但这需要进行一些额外的网络配置，因为您的 OpenWrt 是作为**旁路由**使用的。

当 OpenWrt 是旁路由时，局域网内的电脑（例如 IP 为 `192.168.31.100`）的默认网关通常是您的主路由器（IP 为 `192.168.31.1`），而不是 OpenWrt（例如 IP 为 `192.168.31.9`）。

因此，要实现从 LAN 访问 ZeroTier 客户端，需要确保数据包能够正确地通过 OpenWrt 路由到 ZeroTier 网络。

以下是关键步骤和需要考虑的配置：

1. **在您的主路由器上配置静态路由 (最关键的一步)**
   
   * 这是因为局域网电脑发往 ZeroTier 客户端（例如 `192.168.196.X`，假设这是您的 ZeroTier 网络段）的数据包会首先到达主路由器 (`192.168.31.1`)。主路由器需要知道要把这些数据包转发给谁。
   * 您需要在主路由器（`192.168.31.1`）的管理界面中添加一条静态路由规则：
     * **目标网络 (Destination Network/IP Address):** 您的 ZeroTier 网络的 CIDR 地址（例如 `192.168.196.0`）。
     * **子网掩码 (Subnet Mask):** 您的 ZeroTier 网络的子网掩码（例如 `255.255.255.0`，如果您的 ZT 网络是 /24）。
     * **网关 (Gateway/Next Hop):** 您的 OpenWrt 虚拟机在局域网中的 IP 地址（例如 `192.168.31.9`）。
   * 这样配置后，当主路由器收到目标地址是 ZeroTier 网络的数据包时，它会将其转发给 OpenWrt 处理。
2. **确保 OpenWrt 防火墙允许从 LAN 转发到 ZeroTier 区域**
   
   * 登录 OpenWrt (LuCI 界面)。
   * 导航到 **网络 (Network) -> 防火墙 (Firewall)**。
   * 找到您的 `lan` 防火墙区域，点击 **编辑 (Edit)**。
   * 在 **“区域间的转发” (Inter-Zone Forwarding)** 部分：
     * 确保 **“允许转发到目标区域” (Allow forward to destination zones)** 中包含了您为 ZeroTier 创建的防火墙区域（例如 `zt_vpn` 或 `zerotier`）。
   * 或者，您可以编辑 `zerotier` 区域，确保 **“允许从源区域转发” (Allow forward from source zones)** 中包含了 `lan` 区域。
   * 点击 **保存并应用 (Save & Apply)**。
3. **OpenWrt 上的路由**
   
   * OpenWrt 本身通过其 ZeroTier 接口已经知道如何访问 ZeroTier 网络中的其他客户端，所以这里通常不需要额外配置 OpenWrt 自身的路由表。
4. **ZeroTier 客户端的防火墙**
   
   * 目标 ZeroTier 客户端设备自身的防火墙（例如 Windows 防火墙、Linux 的 iptables/firewalld）需要允许来自您的局域网 IP 地址段 (`192.168.31.0/24`) 的入站连接，或者至少允许您希望访问的特定端口/服务的连接。

**总结一下流程：**

1. 局域网电脑 (`192.168.31.100`) 尝试访问 ZeroTier 客户端 (`192.168.196.X`)。
2. 数据包发送到其默认网关，即主路由器 (`192.168.31.1`)。
3. 主路由器根据其静态路由规则，将数据包转发给 OpenWrt 的 LAN IP (`192.168.31.9`)。
4. OpenWrt 收到数据包，其防火墙允许从 `lan` 区域转发到 `zerotier` 区域。
5. OpenWrt 通过其 ZeroTier 接口将数据包发送到目标 ZeroTier 客户端。
6. ZeroTier 客户端收到数据包（如果其防火墙允许）。

**如果无法配置主路由器怎么办？**

如果您的主路由器不支持配置静态路由，或者您没有权限修改它，那么情况会变得复杂一些：

* **在每台需要访问 ZeroTier 的局域网电脑上手动添加静态路由：**
  * 例如，在 Windows 电脑上：`route add <ZeroTier_Network_CIDR> mask <ZeroTier_Subnet_Mask> <OpenWrt_LAN_IP>`
  * 这种方法管理起来比较麻烦，因为每台电脑都需要单独配置。
* **更复杂的网络配置（如让 OpenWrt 参与 DHCP 或作为主网关的一部分）：** 这通常会改变您旁路由的初衷，并可能需要更深入的网络知识。

**因此，最理想和最干净的方法是在您的主路由器上添加指向 ZeroTier 网络的静态路由，并由 OpenWrt 作为这条路径上的下一跳。**

---
