# iKuai + OpenClash + mini-ppdns 旁路网关部署指南

> **环境定稿**
>
> | 角色 | 地址 | 说明 |
> |------|------|------|
> | iKuai 主路由 | `192.168.9.1` | DHCP / NAT / 静态路由 |
> | OpenWrt (iStoreOS) + OpenClash | `192.168.9.2` | FakeIP / TUN 代理出口 |
> | Ubuntu + mini-ppdns | `192.168.9.3` | 故障转移 DNS |
> | FakeIP 网段 | `198.0.0.0/8` | 虚假 IP 段，路由引流用 |

**实现目标：** 全局透明代理 · FakeIP 分流 · 保留真实客户端源 IP · DHCP 自动下发静态路由 · DNS 与路由解耦

---

## 一、整体网络拓扑

```
                         ┌──────────────────────┐
                         │       Internet        │
                         └──────────┬────────────┘
                                    │
                         ┌──────────▼────────────┐
                         │    iKuai 主路由        │
                         │    192.168.9.1        │
                         │  DHCP / NAT / ACL     │
                         └──────────┬────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
┌─────────▼──────────┐   ┌──────────▼─────────┐   ┌──────────▼─────────┐
│ OpenWrt + OpenClash│   │  Ubuntu mini-ppdns  │   │    LAN Clients     │
│   192.168.9.2      │   │   192.168.9.3       │   │  PC / Phone / NAS  │
│   FakeIP / TUN     │   │   故障转移 DNS       │   │  DHCP from iKuai   │
└─────────┬──────────┘   └──────────┬──────────┘   └──────────┬─────────┘
          │                         │                          │
          │◄──────── DNS 查询 ───────┘                         │
          │                                                    │
          │◄──────────────── FakeIP 流量引流 ──────────────────┘
          │
   ┌──────▼──────┐
   │ OpenClash   │
   │ TUN / Meta  │
   └──────┬──────┘
          │
   ┌──────▼──────┐
   │  代理节点   │
   └─────────────┘
```

---

## 二、网络逻辑说明

### 2.1 DNS 分流工作流

```
客户端发起 DNS 请求
        │
        ▼
mini-ppdns（192.168.9.3:53）
        │
        ├── 国内域名
        │       ├────►默认当 192.168.9.2:7893 请求成功转发至 OpenClash DNS（192.168.9.2:7874）
        │       └────►当 192.168.9.2:7893 请求不成功转发至国内 DNS（223.5.5.5 / 119.29.29.29）
        │                   └── 返回真实 IP
        │
        └── 国外域名 ──► 转发至 OpenClash DNS（192.168.9.2:7874）
                            └── 返回 FakeIP（198.x.x.x）
```

### 2.2 FakeIP 流量工作流

```
客户端访问 www.google.com
        │
        ▼  DNS 返回 198.x.x.x
客户端发起请求至 198.x.x.x
        │
        ▼  iKuai 静态路由：198.0.0.0/8 → 192.168.9.2
OpenClash TUN 接管
        │
        ▼  FakeIP 回查真实域名
代理节点建立连接
        │
        ▼
真实互联网
```

### 2.3 为什么必须配置静态路由

FakeIP 的本质是 **DNS 返回虚假 IP**，`198.x.x.x` 在公网上并不存在。

- **若无静态路由**：客户端访问 `198.x.x.x` → iKuai 按默认路由走 WAN → 公网找不到该 IP → **连接失败**
- **配置静态路由后**：`198.0.0.0/8` 流量强制转发至 `192.168.9.2`（OpenClash）→ TUN 接管 → 正常代理

---

## 三、OpenWrt 端（192.168.9.2）

### 3.1 OpenClash 核心参数

在 OpenClash 管理界面配置如下参数：

| 参数 | 值 |
|------|----|
| 运行模式 | Fake-IP（TUN） |
| Fake-IP 范围 | `198.0.0.0/8` |
| DNS 劫持 | 启用，端口 `7874` |
| SOCKS5 | 启用，端口 `7893` |
| 本地 DNS 劫持 | 启用 |
| IPv6 | 建议关闭 |
| 增强模式 | `fake-ip` |


---

## 四、Ubuntu 端（192.168.9.3）

### 4.1 安装 mini-ppdns

```bash
# 下载二进制
wget -q https://github.com/MetaCubeX/mini-ppdns/releases/latest/download/mini-ppdns-linux-amd64 \
  -O /usr/local/bin/mini-ppdns

# 赋予执行权限
chmod +x /usr/local/bin/mini-ppdns
```

### 4.2 配置文件 `/etc/mini-ppdns.ini`

```ini
[dns]
192.168.9.2:7874

[fall]
223.5.5.5:53
119.29.29.29:53

[listen]
0.0.0.0:53

[adv]
qtime = 250
aaaa = no
lite = yes
boguspriv = 1
```

> **参数说明**
> - `[dns]`：主 DNS，国外域名转发至 OpenClash FakeIP DNS
> - `[fall]`：兜底 DNS，国内域名走国内解析
> - `qtime`：查询超时（ms），超时后降级至 `[fall]`
> - `aaaa = no`：屏蔽 IPv6 解析，防止 AAAA 泄漏
> - `boguspriv = 1`：过滤非法私有地址反向解析

### 4.3 配置 systemd 自启动

创建服务文件：

```bash
nano /etc/systemd/system/mini-ppdns.service
```

写入以下内容：

```ini
[Unit]
Description=mini-ppdns Smart DNS Splitter
After=network-online.target

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

### 4.4 本地验证

```bash
# 查看服务状态
systemctl status mini-ppdns

# 确认 53 端口监听
ss -tlnp | grep :53

# 测试国外域名（应返回 FakeIP）
dig @192.168.9.3 www.google.com
# 预期：www.google.com. IN A 198.x.x.x

# 测试国内域名（应返回真实 IP）
dig @192.168.9.3 www.baidu.com
# 预期：www.baidu.com. IN A 110.x.x.x
```

---

## 五、iKuai 端（192.168.9.1）

### 5.1 DHCP 服务配置

**路径：** 网络设置 → DHCP 设置 → DHCP 服务端 → 编辑 lan1

| 字段 | 值 |
|------|----|
| 首选 DNS | `192.168.9.3` |
| 备选 DNS | `223.5.5.5`（可选，作灾备） |
| 网关 | `192.168.9.1` |
| 租期 | `86400`（秒） |

操作：**保存 → 应用 → 重启 DHCP 服务**

### 5.2 静态路由（FakeIP 引流核心）

**路径：** 网络设置 → 路由设置 → 静态路由 → 添加

| 字段 | 值 |
|------|----|
| 目的地址 | `198.0.0.0` |
| 子网掩码 | `255.0.0.0`（即 /8） |
| 网关 | `192.168.9.2` |
| 接口 | `LAN1` |
| 优先级 | `1` |
| 描述 | `OpenClash-FakeIP` |

操作：**保存 → 应用**，验证路由表中 `198.0.0.0` 状态为已启用。

### 5.3 DHCP Option 121（自动下发路由至客户端）

**路径：** DHCP 服务端编辑 → 高级设置 → 自定义选项

| 字段 | 值 |
|------|----|
| Option | `121` |
| 类型 | 字符串 |
| 值 | `198.0.0.0/8 192.168.9.2` |

备选 Hex 写法：`08 C6 00 00 00 00 00 00 00 02`

> **作用**：客户端通过 DHCP 续租时自动获得 `198.0.0.0/8 → 192.168.9.2` 的主机路由，无需手动添加。Windows / Linux / Android 通常均可自动接收。

### 5.4 NAT 过滤（保留真实客户端源 IP）

**路径：** 网络设置 → NAT 规则 → NAT 过滤 → 添加

| 字段 | 值 |
|------|----|
| 源地址 | `192.168.9.0/24` |
| 目的地址 | `198.0.0.0/8` |
| 动作 | **不 NAT** |
| 方向 | 转发 |
| 描述 | `FakeIP-No-NAT` |

> ⚠️ **关键：保存后必须将此规则拖至列表最顶部**，否则会被默认 NAT 规则覆盖。
>
> **效果**：OpenClash 日志中显示的是真实客户端 IP（如 `192.168.9.100`），而非主路由 IP（`192.168.9.1`）。

### 5.5 ACL 放行（如有访问控制拦截）

**路径：** 网络安全 → ACL 规则 → 添加

| 规则 | 源 | 目的 | 端口 | 动作 |
|------|----|------|------|------|
| 规则 1 | LAN | `192.168.9.2` | 全部 | 允许 |
| 规则 2 | LAN | `192.168.9.3` | `53` | 允许 |

> ⚠️ 以上规则必须置于所有**拒绝规则之前**。

---

## 六、重启顺序

按以下顺序依次重启，确保依赖链正确建立：

```
1. Ubuntu       → systemctl restart mini-ppdns
2. OpenWrt      → 重启 OpenClash 服务
3. iKuai        → 重启网络服务（或整机重启）
4. 客户端       → ipconfig /release && ipconfig /renew
```

---

## 七、验证清单

| 验证项 | 命令 | 预期结果 |
|--------|------|----------|
| DNS 返回 FakeIP | `nslookup www.google.com 192.168.9.3` | `198.x.x.x` |
| 国内 DNS 正常 | `nslookup www.baidu.com 192.168.9.3` | 国内真实 IP |
| 路由指向正确 | `tracert -d 198.0.0.1` | 第一跳为 `192.168.9.2` |
| TUN 接口存在 | `ip a \| grep utun` | 显示 `utun` 接口 |
| 源 IP 保留 | 查看 OpenClash 连接日志 | 显示客户端真实 IP，非 `192.168.9.1` |
| DNS 灾备生效 | 停止 mini-ppdns 后测试百度解析 | 仍可解析（需备用 DNS 配置正确） |

---

## 八、故障排查树

```
无法科学上网
│
├── DNS 是否返回 FakeIP？
│      ├── 否 ──► mini-ppdns 未运行
│      │         OpenClash DNS 未监听 7874
│      │         上游 DNS 配置错误
│      └── 是
│
├── 客户端能否路由到 198.x.x.x？
│      ├── 否 ──► iKuai 静态路由未配置
│      │         DHCP Option 121 未生效
│      │         ACL 规则拦截
│      └── 是
│
├── OpenClash TUN 是否接管流量？
│      ├── 否 ──► TUN 接口未启动（检查 utun）
│      │         FakeIP 网段与其他网段冲突
│      │         auto-route 配置错误
│      └── 是
│
└── 代理是否正常出口？
       ├── 否 ──► 节点失效或超时
       │         分流规则配置错误
       │         FakeIP 域名回查失败
       └── 是 ──► 系统工作正常 ✓
```

---

## 九、进阶优化建议

| 方向 | 建议 | 解决的问题 |
|------|------|------------|
| 网络隔离 | 为 FakeIP 流量划独立 VLAN | 避免与业务 LAN 混杂 |
| 接口隔离 | OpenClash 使用独立网卡 | 防止桥接转发异常 |
| IPv6 处理 | 全局禁用 IPv6 | 避免 AAAA 泄漏绕过代理 |
| 多策略 DNS | mini-ppdns 多实例部署 | 不同 VLAN 使用不同 DNS 策略 |
| 复杂路由 | OpenClash TUN 独立策略路由 | 适配多 WAN 复杂环境 |
