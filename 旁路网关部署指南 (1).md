# iKuai + OpenClash + mini-ppdns 旁路网关部署指南

> **环境定稿**
>
> | 角色 | 地址 | 说明 |
> |------|------|------|
> | iKuai 主路由 | `192.168.9.1` | DHCP / NAT / 静态路由 |
> | OpenWrt (iStoreOS) + OpenClash | `192.168.9.2` | FakeIP / TUN 代理出口 |
> | Ubuntu + mini-ppdns | `192.168.9.3` | 主 DNS / 故障转移 |
> | FakeIP 网段 | `198.18.0.0/15` | 虚假 IP 段，路由引流用 |

> **关于 FakeIP 网段选择**
>
> 原始教程使用 `198.0.0.0/8`，该段过于宽泛（覆盖 1600 万地址），且部分地址（如 `198.51.100.0/24`）属于 IANA 文档保留段，可能与真实流量产生冲突。推荐使用 Clash Meta 默认的 `198.18.0.0/15`（IANA 基准测试保留段，共 13.1 万地址，不会出现在公网路由中）。本文统一使用 `198.18.0.0/15`，若你的配置已使用 `/8`，需保持路由、DHCP Option 121、OpenClash FakeIP 范围三处一致。

**实现目标：** 全局透明代理 · FakeIP 分流 · 保留真实客户端源 IP · DHCP 自动下发静态路由 · DNS 与路由解耦

---

## 一、整体网络拓扑

### 1.1 物理 / 逻辑连接

所有设备都挂在同一个 LAN（`192.168.9.0/24`）下，iKuai 是唯一网关，负责上网出口、DHCP 分配和静态路由。OpenWrt 和 Ubuntu 作为旁路设备，不承担网关职责，客户端的默认网关始终是 iKuai。

```
  互联网
    │
    │ WAN
┌───┴──────────────────────────────────────────────────┐
│  iKuai 主路由  192.168.9.1                           │
│  · DHCP 服务（下发 DNS=9.3，Option 121 静态路由）     │
│  · 静态路由：198.18.0.0/15 → 192.168.9.2            │
│  · NAT 配置：开启 LAN→LAN NAT，FakeIP 段设不 NAT 例外 │
└───┬──────────────────────────────────────────────────┘
    │
    │ LAN  192.168.9.0/24
    ├────────────────────┬────────────────────┐
    │                    │                    │
┌───┴────────────┐  ┌────┴────────────┐  ┌───┴──────────────────┐
│ OpenWrt        │  │ Ubuntu          │  │ 客户端               │
│ 192.168.9.2    │  │ 192.168.9.3     │  │ PC / 手机 / NAS 等   │
│                │  │                 │  │                      │
│ · OpenClash    │  │ · mini-ppdns    │  │ · 网关 → 9.1         │
│   TUN 接管     │  │   :53 监听      │  │ · DNS  → 9.3         │
│   FakeIP 解析  │  │   主 DNS / 故障  │  │ · 路由表自动获得     │
│   代理出口     │  │   转移          │  │   198.18.0.0/15→9.2  │
└───────────────┘  └────────────────┘  └──────────────────────┘
```

### 1.2 数据流向

同一次访问涉及两条独立的流：**DNS 查询流**（决定返回真实 IP 还是 FakeIP）和**数据流**（根据 IP 决定走直连还是代理）。

**① DNS 查询流**

```
客户端
  │  DNS 查询 www.google.com
  ▼
mini-ppdns  192.168.9.3:53
  │  全量转发（不做域名分流，分流由 OpenClash 负责）
  ▼
OpenClash DNS  192.168.9.2:7874
  │
  ├─ fake-ip-filter 命中 real-ip（国内域名、局域网等）
  │     └─► 返回真实 IP  ──► 客户端直连，流量走 iKuai 正常上网
  │
  └─ fake-ip-filter 命中 fake-ip（或 MATCH 兜底）
        └─► 返回 FakeIP 198.18.x.x  ──► 进入 ② 数据流
```

**② FakeIP 数据流**

```
客户端
  │  目的 IP = 198.18.x.x（FakeIP，公网不存在）
  ▼
iKuai 路由表
  │  静态路由命中：198.18.0.0/15 → 192.168.9.2
  │  NAT 过滤：此段流量不做 SNAT，源 IP 保持客户端真实 IP
  ▼
OpenClash TUN  192.168.9.2
  │  根据 FakeIP ↔ 域名映射表，还原真实域名
  │  按代理规则选择节点
  ▼
代理节点
  │
  ▼
互联网目标服务器
```

**③ hook 故障探测流（后台独立运行）**

```
mini-ppdns（每 60 秒）
  │  curl 通过 socks5h://192.168.9.2:7893 访问 Google 204
  │
  ├─ 成功（返回 HTTP 204）→ 继续使用 OpenClash DNS，无感知
  │
  └─ 连续 10 次失败 → 判定代理链路故障
        └─► 自动切换全部 DNS 至备用（223.5.5.5）
              代理失效期间客户端走直连，国内访问不受影响
              OpenClash 恢复后自动切回
```

---

## 二、OpenWrt 端（192.168.9.2）

### 2.1 OpenClash 核心参数

在 OpenClash 管理界面配置如下参数：

| 参数 | 值 | 说明 |
|------|----|------|
| 运行模式 | Fake-IP（TUN） | 必须选 TUN 模式，否则无法接管三层流量 |
| Fake-IP 范围 | `198.18.0.0/15` | 与 iKuai 静态路由、DHCP Option 121 保持一致 |
| DNS 监听端口 | `7874` | mini-ppdns 主 DNS 指向此端口 |
| SOCKS5 端口 | `7893` | mini-ppdns hook 探活使用此端口 |
| 本地 DNS 劫持 | 启用 | 确保客户端 DNS 请求不绕过 OpenClash |
| IPv6 | 关闭 | 避免 AAAA 记录泄漏绕过代理 |
| 增强模式 | `fake-ip` | |

### 2.2 fake-ip-filter 精细规则模式

Mihomo 新版支持 `fake-ip-filter-mode: rule`，可以用规则集（RULE-SET）精细控制每个域名/域名集合返回真实 IP 还是 FakeIP，比旧版纯域名白名单列表灵活得多。

规则按**从上到下首次命中**执行，末尾 `MATCH,fake-ip` 作为兜底，未匹配的域名全部走代理：

```yaml
dns:
  enhanced-mode: fake-ip
  fake-ip-range: 198.18.0.0/15
  fake-ip-filter-mode: rule        # 启用规则模式（Mihomo 新版）
  fake-ip-filter:
    # ── 强制返回真实 IP（直连域名）──────────────────────────────
    - RULE-SET,fakeip-filter,real-ip   # Mihomo 官方维护的直连过滤集（NTP、STUN 等）
    - DOMAIN,mimimi.com,real-ip        # 单个域名直连（按需添加）
    - RULE-SET,LocationDKS,real-ip     # 本地/家庭服务
    - RULE-SET,Private,real-ip         # 局域网 / RFC1918 地址
    - RULE-SET,Direct,real-ip          # 自定义直连规则集
    - RULE-SET,XPTV,real-ip            # XPTV 本地直连
    - RULE-SET,Download,real-ip        # 下载类（BT/迅雷等）走直连
    - RULE-SET,AppleCN,real-ip         # Apple 国内 CDN（直连更快）
    - RULE-SET,China,real-ip           # 国内域名合集（最重要，防止国内网站返回 FakeIP）
    # ── 强制返回 FakeIP（代理域名）──────────────────────────────
    - RULE-SET,AI,fake-ip              # AI 服务（OpenAI / Claude 等）
    - DOMAIN-KEYWORD,speedtest,fake-ip # 含 speedtest 关键字的域名走代理测速
    - RULE-SET,Speedtest,fake-ip
    - RULE-SET,Twitter,fake-ip
    - RULE-SET,Telegram,fake-ip
    - RULE-SET,SocialMedia,fake-ip
    - RULE-SET,NewsMedia,fake-ip
    - RULE-SET,Games,fake-ip
    - RULE-SET,Crypto,fake-ip
    - RULE-SET,Emby,fake-ip
    - RULE-SET,Netflix,fake-ip
    - RULE-SET,YouTube,fake-ip
    - RULE-SET,Streaming,fake-ip
    - RULE-SET,Apple,fake-ip           # Apple 国际服务（区别于上面的 AppleCN）
    - RULE-SET,Google,fake-ip
    - RULE-SET,Microsoft,fake-ip
    - RULE-SET,Proxy,fake-ip           # 其他需代理的规则集
    # ── 兜底：未匹配的全部走代理 ────────────────────────────────
    - MATCH,fake-ip
```

> **注意事项**
> - `China,real-ip` 一定要放在 `MATCH,fake-ip` 之前，否则国内域名也会返回 FakeIP，导致国内访问全部绕路代理
> - `AppleCN` 与 `Apple` 分别对应国内 CDN 和国际服务，两者同时存在时 `AppleCN` 须写在 `Apple` 之前（先命中先生效）
> - `fakeip-filter` 官方规则集包含 NTP、STUN、mDNS 等必须直连的协议，建议始终保留

---

## 三、Ubuntu 端（192.168.9.3）

### 3.1 安装 mini-ppdns

```bash
# 下载二进制（以 Ubuntu x86_64 为例）
wget -q https://github.com/kkkgo/mini-ppdns/raw/refs/heads/release/mini-ppdns_x86_64 \
  -O /usr/local/bin/mini-ppdns

# 赋予执行权限
chmod +x /usr/local/bin/mini-ppdns
```

> 若为 ARM 架构（如树莓派），请下载对应的 `mini-ppdns_aarch64`。

### 3.2 配置文件 `/etc/mini-ppdns.ini`

以下为本场景推荐的最简配置，去除了与本方案无关的示例注释：

```ini
# 主 DNS：所有 DNS 请求优先发往 OpenClash FakeIP DNS
[dns]
192.168.9.2:7874

# 备用/故障转移 DNS：主 DNS 不可达时自动切换
[fall]
223.5.5.5:53
119.29.29.29:53

# 监听地址端口
# 指定 0.0.0.0 时会自动展开为本机所有私有/环回地址，不对外暴露公网接口
[listen]
127.0.0.1:53
192.168.9.3:53

# [force_fall] 可选：强制让指定 IP 段的设备走备用 DNS（跳过代理）
# 本场景默认不启用，如需特定设备不走代理可在此配置
# [force_fall]
# 192.168.9.100          # 单个 IP，该设备 DNS 始终走运营商
# 192.168.9.200/30       # CIDR 段
# ^192.168.9.0/24        # 取反：除此之外的设备都走备用 DNS（谨慎使用）

[adv]
# 切换延迟阈值（毫秒）：主 DNS 响应超过此值视为超时，尝试备用
qtime=250
# IPv6 AAAA 记录：关闭，防止 IPv6 流量绕过代理
aaaa=no
# 精简响应模式：去掉无关记录（推荐开启）
lite=yes

# Hook：通过探测 SOCKS5 代理可用性，决定是否切换到备用 DNS
[hook]
# 通过 OpenClash 的 SOCKS5 端口访问 Google，检测代理是否正常
exec="curl -o /dev/null -s -w %{http_code} --proxy socks5h://192.168.9.2:7893 http://www.google.com/generate_204"
# 命令退出码为 0 视为成功
exit_code=0
# 响应体包含 "204" 视为成功
keyword="204"
# 每 60 秒探测一次
sleep_time=60
# 探测失败后 5 秒重试
retry_time=5
# 连续 10 次失败后才判定主 DNS 故障（防止短暂抖动误切换）
count=10
# 切换至备用 DNS 时执行（可选，用于告警通知）
# switch_fall_exec="curl -sk -o /dev/null --data 'Main DNS is DOWN!' --retry 3 https://ntfy.sh/mydns_status"
# 切换回主 DNS 时执行（可选）
# switch_main_exec="curl -sk -o /dev/null --data 'Main DNS is UP!' --retry 3 https://ntfy.sh/mydns_status"
```

> **配置说明**
> - `[dns]`：主 DNS，所有请求先到 OpenClash，由其负责 FakeIP/直连分流
> - `[fall]`：兜底 DNS，仅在 hook 判定主 DNS 故障后生效
> - `[hook]`：以 SOCKS5 代理访问 Google 204 接口作为存活探测，相比直接 ping OpenClash 更能反映代理链路的真实可用性

### 3.3 配置 systemd 自启动

创建服务文件：

```bash
nano /etc/systemd/system/mini-ppdns.service
```

写入以下内容：

```ini
[Unit]
Description=mini-ppdns Smart DNS Splitter
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mini-ppdns -config /etc/mini-ppdns.ini -d
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

启用并启动：

```bash
systemctl daemon-reload
systemctl enable --now mini-ppdns
```

### 3.4 本地验证

```bash
# 查看服务状态
systemctl status mini-ppdns

# 确认 53 端口监听
ss -tlnp | grep :53

# 测试国外域名（应返回 FakeIP：198.18.x.x）
dig @192.168.9.3 www.google.com A
# 预期：www.google.com. IN A 198.18.x.x

# 测试国内域名（应返回真实 IP，前提是已正确配置 fake-ip-filter）
dig @192.168.9.3 www.baidu.com A
# 预期：www.baidu.com. IN A 110.x.x.x 等国内真实 IP

# 测试 hook 探活（手动执行，验证 curl 命令是否正常）
curl -o /dev/null -s -w "%{http_code}\n" \
  --proxy socks5h://192.168.9.2:7893 \
  http://www.google.com/generate_204
# 预期输出：204
```

---

## 四、iKuai 端（192.168.9.1）

### 4.1 DHCP 服务配置

**路径：** 网络设置 → DHCP 设置 → DHCP 服务端 → 编辑 lan1

| 字段 | 值 | 说明 |
|------|----|------|
| 首选 DNS | `192.168.9.3` | 指向 mini-ppdns |
| 备选 DNS | `223.5.5.5`（可选） | 仅作灾备，正常情况下 mini-ppdns 内部已有 fallback |
| 网关 | `192.168.9.1` | 保持默认，网关不变 |
| 租期 | `86400`（秒） | |

操作：**保存 → 应用 → 重启 DHCP 服务**

### 4.2 静态路由（FakeIP 引流核心）

**路径：** 网络设置 → 路由设置 → 静态路由 → 添加

| 字段 | 值 |
|------|----|
| 目的地址 | `198.18.0.0` |
| 子网掩码 | `255.254.0.0`（即 /15） |
| 网关 | `192.168.9.2` |
| 接口 | `LAN1` |
| 优先级 | `1` |
| 描述 | `OpenClash-FakeIP` |

操作：**保存 → 应用**，验证路由表中 `198.18.0.0` 状态为已启用。

### 4.3 DHCP Option 121（自动下发路由至客户端）

**路径：** DHCP 服务端编辑 → 高级设置 → 自定义选项

Option 121 支持在一条记录中同时下发多条路由，用空格分隔，格式为 `网段/前缀 网关 网段/前缀 网关 ...`：

| 字段 | 值 |
|------|----|
| Option | `121` |
| 类型 | 字符串（文本格式） |
| 值 | `198.18.0.0/15 192.168.9.2 0.0.0.0/0 192.168.9.1` |

两条路由的含义：

| 路由条目 | 作用 |
|----------|------|
| `198.18.0.0/15 → 192.168.9.2` | FakeIP 流量引流至 OpenClash |
| `0.0.0.0/0 → 192.168.9.1` | 默认路由，指向 iKuai 主路由 |

> **为什么要加默认路由？**
>
> RFC 3442 规定：客户端若收到 Option 121，**必须忽略** Option 3（传统默认网关字段）。部分设备（已知包括小米/米家 IoT 设备）严格遵守此规定，如果 Option 121 中没有包含默认路由 `0.0.0.0/0`，这些设备收到 DHCP 后将没有默认路由，导致无法正常上网。显式写入 `0.0.0.0/0 192.168.9.1` 可确保所有设备兼容。

> **Hex 写法**（若 iKuai 需要原始 Hex 格式）：
>
> Option 121 编码规则：每条路由为 `前缀长度(1字节) + 网络地址有效字节 + 网关地址(4字节)`，多条路由直接拼接。
>
> - `198.18.0.0/15`：前缀 `0F`，网络地址有效字节 `C6 12`，网关 `C0 A8 09 02`
> - `0.0.0.0/0`：前缀 `00`，网络地址有效字节为空（/0 无需填写），网关 `C0 A8 09 01`
>
> ```
> 0F C6 12 C0 A8 09 02 00 C0 A8 09 01
> ```

### 4.4 NAT 配置（保留真实客户端源 IP）

本方案中客户端流量从 LAN（`192.168.9.x`）转发至同在 LAN 的 OpenClash（`192.168.9.2`），属于 **LAN→LAN 转发**，需要在两处配置 NAT：

**① iKuai 4.0：开启 LAN→LAN NAT**

iKuai 4.0 默认过滤了 LAN 到 LAN 的 NAT，使用下一跳旁路网关分流时必须手动开启，否则 OpenClash 收到的数据包源 IP 会异常，导致回包无法正确返回客户端。

**路径：** 网络设置 → NAT 规则 → NAT 设置

找到"LAN to LAN NAT"或"局域网间 NAT"选项，将其**开启**并保存应用。

> 若 iKuai 版本较早（4.0 以前），此选项默认开启，可跳过此步。

**② 添加 NAT 过滤规则（不 NAT，保留源 IP）**

开启 LAN→LAN NAT 后，iKuai 默认会对所有 LAN 转发流量做 SNAT，导致 OpenClash 看到的来源 IP 全是 `192.168.9.1`（主路由），而非真实客户端 IP。需添加"不 NAT"例外规则，让 FakeIP 流量保留真实源 IP。

**路径：** 网络设置 → NAT 规则 → NAT 过滤 → 添加

| 字段 | 值 |
|------|----|
| 源地址 | `192.168.9.0/24` |
| 目的地址 | `198.18.0.0/15` |
| 动作 | **不 NAT** |
| 方向 | 转发 |
| 描述 | `FakeIP-No-NAT` |

> ⚠️ **关键：保存后必须将此规则拖至列表最顶部**，否则会被默认 NAT 规则覆盖。
>
> **效果**：OpenClash 日志中显示的是真实客户端 IP（如 `192.168.9.100`），而非主路由 IP（`192.168.9.1`）。这对按客户端分流、审计日志、限速策略均有重要意义。

### 4.5 ACL 放行（如有访问控制拦截）

**路径：** 网络安全 → ACL 规则 → 添加

| 规则 | 源 | 目的 | 端口 | 动作 |
|------|----|------|------|------|
| 规则 1 | LAN | `192.168.9.2` | 全部 | 允许 |
| 规则 2 | LAN | `192.168.9.3` | `53` | 允许 |

> ⚠️ 以上规则必须置于所有**拒绝规则之前**，否则 DNS 和代理流量会被 ACL 静默丢弃。

---

## 五、重启顺序

按以下顺序依次重启，确保依赖链正确建立：

```
1. Ubuntu       → systemctl restart mini-ppdns
2. OpenWrt      → 重启 OpenClash 服务
3. iKuai        → 重启网络服务（或整机重启）
4. 客户端       → Windows: ipconfig /release && ipconfig /renew
                   Linux:   dhclient -r && dhclient
                   Android/iOS: 关闭 Wi-Fi 再重连
```

> 重启顺序至关重要：OpenClash 的 DNS 服务（7874）必须先于 mini-ppdns 就绪，否则 mini-ppdns 启动后首次探测主 DNS 可能因超时直接切换至备用 DNS。

---

## 六、验证清单

| 验证项 | 命令 | 预期结果 |
|--------|------|----------|
| DNS 返回 FakeIP | `nslookup www.google.com 192.168.9.3` | `198.18.x.x` |
| 国内 DNS 正常 | `nslookup www.baidu.com 192.168.9.3` | 国内真实 IP |
| 路由指向正确 | `tracert -d 198.18.0.1`（Windows）<br>`traceroute -n 198.18.0.1`（Linux） | 第一跳为 `192.168.9.2` |
| TUN 接口存在 | `ip link show`（OpenWrt SSH） | 显示 `tun` 或 `utun` 接口 Up 状态 |
| 源 IP 保留 | 查看 OpenClash 连接日志 | 显示客户端真实 IP，非 `192.168.9.1` |
| hook 探活正常 | `journalctl -u mini-ppdns -f` | 日志显示 hook 检测成功，无切换事件 |
| DNS 灾备生效 | 停止 OpenClash DNS，等待 hook 触发后测试解析 | 自动切至 `223.5.5.5`，百度可正常解析 |

---

## 七、故障排查树

```
无法科学上网
│
├── DNS 是否返回 FakeIP？
│      ├── 否 ──► mini-ppdns 是否运行？（systemctl status mini-ppdns）
│      │          OpenClash DNS 是否监听 7874？（ss -tlnp | grep 7874，在 OpenWrt 上执行）
│      │          mini-ppdns 是否已切换至备用 DNS？（journalctl -u mini-ppdns | grep hook）
│      └── 是
│
├── 客户端能否路由到 198.18.x.x？
│      ├── 否 ──► iKuai 静态路由是否生效？（路由表检查 198.18.0.0）
│      │          DHCP Option 121 是否下发？（客户端执行 ip route 或 route print 检查）
│      │          ACL 规则是否拦截了 FakeIP 目的流量？
│      └── 是
│
├── OpenClash TUN 是否接管流量？
│      ├── 否 ──► TUN 接口是否存在且 Up？（ip link show，在 OpenWrt SSH 执行）
│      │          FakeIP 网段与其他网段是否冲突？
│      │          OpenClash auto-route / auto-redirect 是否配置正确？
│      └── 是
│
└── 代理是否正常出口？
       ├── 否 ──► 节点是否失效或超时？（OpenClash 控制台查看延迟）
       │          分流规则是否配置错误？
       │          fake-ip-filter 是否将目标域名错误地排除？
       └── 是 ──► 系统工作正常 ✓
```

---

## 八、进阶优化建议

| 方向 | 建议 | 解决的问题 |
|------|------|------------|
| 国内域名覆盖 | 在 `fake-ip-filter` 的 `China,real-ip` 规则集中引入完整国内域名库（如 [Loyalsoldier/clash-rules](https://github.com/Loyalsoldier/clash-rules)） | 防止国内域名误返回 FakeIP |
| 网络隔离 | 为 FakeIP 流量划独立 VLAN | 避免与业务 LAN 混杂 |
| 接口隔离 | OpenClash 使用独立网卡 | 防止桥接转发异常 |
| IPv6 彻底禁用 | iKuai、OpenWrt、Ubuntu 三端均关闭 IPv6 | 避免 AAAA 泄漏绕过代理 |
| 告警通知 | 配置 mini-ppdns 的 `switch_fall_exec` / `switch_main_exec` | 代理链路故障时及时感知 |
| 多设备策略 | 利用 `force_fall` 对特定客户端 IP 强制走直连 DNS | 实现设备级粒度的代理豁免 |
| 复杂路由 | OpenClash TUN 独立策略路由 | 适配多 WAN 复杂环境 |
