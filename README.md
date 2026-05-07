iKuai + OpenClash + mini-ppdns 旁路网关实录

> 基于：

iKuai 主路由

OpenWrt(iStoreOS) + OpenClash Fake-IP(TUN)

Ubuntu + mini-ppdns


实现：

全局透明代理

FakeIP 分流

保留真实客户端源 IP

DHCP 自动下发静态路由

DNS 与路由解耦





---

一、整体网络拓扑

┌──────────────────────┐
                                      │      Internet        │
                                      └─────────┬────────────┘
                                                │
                                      ┌─────────▼────────────┐
                                      │      iKuai 主路由     │
                                      │      192.168.9.1     │
                                      │  DHCP / NAT / ACL    │
                                      └─────────┬────────────┘
                                                │
                           ┌────────────────────┼────────────────────┐
                           │                    │                    │
                           │                    │                    │
                ┌──────────▼─────────┐ ┌────────▼─────────┐ ┌────────▼─────────┐
                │ OpenWrt+iStoreOS   │ │ Ubuntu Server    │ │    LAN Clients   │
                │ OpenClash          │ │ mini-ppdns       │ │ PC/Phone/NAS     │
                │ 192.168.9.2        │ │ 192.168.9.3      │ │ DHCP from iKuai  │
                │ FakeIP/TUN         │ │ Smart DNS Split  │ │                  │
                └──────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
                           │                    │                    │
                           │                    │                    │
                           │◄──── DNS 查询 ─────┘                    │
                           │                                          │
                           │◄──── FakeIP 流量引流 ────────────────────┘
                           │
                    ┌──────▼──────┐
                    │ OpenClash   │
                    │ TUN/Meta内核 │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ 代理节点出口 │
                    └─────────────┘


---

二、网络逻辑说明


---

2.1 DNS 工作流

客户端
   │
   │ DNS 请求
   ▼
mini-ppdns（192.168.9.3）
   │
   ├── 国内域名
   │      └── 返回真实 IP
   │
   └── 国外域名
          └── 转发给 OpenClash DNS(7874)
                     │
                     └── 返回 FakeIP
                             198.x.x.x


---

2.2 FakeIP 流量工作流

客户端访问：
www.google.com → 198.x.x.x

客户端
   │
   ▼
iKuai
   │
   │ 静态路由:
   │ 198.0.0.0/8 → 192.168.9.2
   ▼
OpenClash TUN
   │
   │ FakeIP 回查真实域名
   ▼
代理节点
   │
   ▼
真实互联网


---

2.3 为什么必须使用静态路由

FakeIP 本质：

DNS 返回的是假的 IP
198.x.x.x 并不存在于公网

如果：

没有静态路由

那么：

客户端访问 198.x.x.x
↓
iKuai 默认走 WAN
↓
公网不存在
↓
连接失败

因此必须：

198.0.0.0/8
↓
强制送到 OpenClash


---

三、OpenWrt 端（192.168.9.2）


---

3.1 OpenClash 模式设定

OpenClash 推荐参数

运行模式:
  Fake-IP（TUN）

Fake-IP 范围:
  198.0.0.0/8

DNS 劫持:
  启用
  端口: 7874

SOCKS5:
  启用
  端口: 7893

本地 DNS 劫持:
  启用

IPv6:
  建议关闭

增强模式:
  fake-ip

Tun:
  stack: system
  auto-route: false
  auto-detect-interface: true


---

3.2 内核 IP 转发

临时开启

echo 1 > /proc/sys/net/ipv4/ip_forward

永久开启

echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

验证

sysctl net.ipv4.ip_forward

预期：

net.ipv4.ip_forward = 1


---

3.3 确认 TUN 设备

查看 utun

ip a | grep utun

预期：

utun: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP>


---

检查端口监听

ss -tlnp | grep -E '7874|7893'

预期：

LISTEN 0 4096 *:7874
LISTEN 0 4096 *:7893


---

四、Ubuntu 端（192.168.9.3）


---

4.1 安装 mini-ppdns

下载

wget -q https://github.com/MetaCubeX/mini-ppdns/releases/latest/download/mini-ppdns-linux-amd64 \
-O /usr/local/bin/mini-ppdns

赋权

chmod +x /usr/local/bin/mini-ppdns


---

4.2 配置文件 /etc/mini-ppdns.ini

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


---

4.3 写入 systemd 自启

创建服务

nano /etc/systemd/system/mini-ppdns.service

服务内容

[Unit]
Description=mini-ppdns
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mini-ppdns -config /etc/mini-ppdns.ini -d
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target


---

重载并启动

systemctl daemon-reload
systemctl enable --now mini-ppdns


---

4.4 本地验证

查看状态

systemctl status mini-ppdns


---

查看 53 端口

ss -tlnp | grep :53


---

测试 FakeIP

dig @192.168.9.3 www.google.com

预期：

www.google.com.  IN A 198.x.x.x


---

测试国内解析

dig @192.168.9.3 www.baidu.com

预期：

www.baidu.com. IN A 110.x.x.x


---

五、iKuai 端（192.168.9.1）


---

5.1 DHCP 服务

路径：

网络设置
 └── DHCP 设置
      └── DHCP 服务端
           └── 编辑 lan1

配置：

首选 DNS:
192.168.9.3

备选 DNS:
223.5.5.5（可选）

网关:
192.168.9.1

租期:
86400

操作：

保存
→ 应用
→ 重启 DHCP 服务


---

5.2 静态路由（FakeIP 引流）

路径：

网络设置
 └── 路由设置
      └── 静态路由

添加：

目的地址:
198.0.0.0

子网掩码:
255.0.0.0

网关:
192.168.9.2

接口:
LAN1

优先级:
1

描述:
OpenClash-FakeIP


---

路由逻辑

198.0.0.0/8
↓
OpenClash
↓
TUN 接管
↓
代理


---

5.3 DHCP Option 121（自动下发静态路由）

路径：

DHCP 服务端编辑
 └── 高级设置
      └── 自定义选项

配置：

Option:
121

类型:
字符串

值:
198.0.0.0/8 192.168.9.2

Hex 写法：

08 C6 00 00 00 00 00 00 00 02


---

作用

客户端无需手动加路由：

DHCP 自动下发：
198.0.0.0/8 → 192.168.9.2

Windows / Linux / Android 通常可自动接收。


---

5.4 NAT 过滤（保源 IP）

路径：

网络设置
 └── NAT 规则
      └── NAT 过滤

添加：

源地址:
192.168.9.0/24

目的地址:
198.0.0.0/8

动作:
不 NAT

方向:
转发

描述:
FakeIP-No-NAT


---

关键

必须拖到最顶部。

否则：

会被默认 NAT 覆盖


---

作用

避免：

OpenClash 日志看到：
192.168.9.1

而是：

真实客户端 IP

例如：

192.168.9.100
192.168.9.101


---

5.5 ACL 放行（如有阻断）

路径：

网络安全
 └── ACL 规则

添加：

规则1:
源 LAN
目的 192.168.9.2
动作 允许

规则2:
源 LAN
目的 192.168.9.3
端口 53
动作 允许

必须位于：

所有拒绝规则之前


---

六、重启顺序


---

1. Ubuntu
   systemctl restart mini-ppdns

2. OpenWrt
   重启 OpenClash

3. iKuai
   重启网络服务

4. 客户端
   ipconfig /release
   ipconfig /renew


---

七、验证清单

验证项	命令	预期

DNS 发 FakeIP	nslookup www.google.com 192.168.9.3	198.x.x.x
国内解析	nslookup www.baidu.com 192.168.9.3	国内真实 IP
路由正确	tracert -d 198.0.0.1	第一跳 192.168.9.2
TUN 存在	ip a | grep utun	utun 接口存在
源 IP 不丢失	OpenClash 日志	显示真实客户端 IP
DNS 灾备	停止上游后测试	国内解析仍可用



---

八、完整故障排查树

无法科学上网
│
├── DNS 是否返回 FakeIP？
│      │
│      ├── 否
│      │    ├── mini-ppdns 未运行
│      │    ├── OpenClash DNS 未监听 7874
│      │    └── 上游 DNS 配置错误
│      │
│      └── 是
│
├── 客户端能否访问 198.x.x.x？
│      │
│      ├── 否
│      │    ├── iKuai 无静态路由
│      │    ├── DHCP Option121 未生效
│      │    └── ACL 阻断
│      │
│      └── 是
│
├── OpenClash 是否接管？
│      │
│      ├── 否
│      │    ├── TUN 未启动
│      │    ├── FakeIP 网段冲突
│      │    └── auto-route 配置错误
│      │
│      └── 是
│
└── 是否成功代理？
       │
       ├── 否
       │    ├── 节点失效
       │    ├── 分流规则错误
       │    └── DNS 回查失败
       │
       └── 是
            └── 系统正常


---

九、最终效果

实现：

DNS：
国内直连
国外 FakeIP

流量：
国外流量自动进 OpenClash

日志：
保留真实客户端源 IP

客户端：
无需手动配置代理

系统：
旁路由透明接管


---

十、推荐增强项（进阶）


---

建议增加：

1. FakeIP 独立 VLAN

避免：

FakeIP 与业务 LAN 混杂


---

2. OpenClash 独立网卡

避免：

桥接转发异常


---

3. DNS 双栈隔离

建议：

IPv6 全禁

否则：

AAAA 泄漏


---

4. mini-ppdns 多实例

可实现：

不同 VLAN 不同 DNS 策略


---

5. OpenClash TUN 独立策略路由

适合：
复杂多 WAN环境。
