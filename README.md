环境定稿：
- iKuai 主路由：`192.168.9.1`
- OpenWrt(iStoreOS) + OpenClash：`192.168.9.2`
- Ubuntu + mini-ppdns：`192.168.9.3`
- FakeIP 网段：`198.0.0.0/8`

---

一、OpenWrt 端（192.168.9.2）

1.1 OpenClash 模式设定

[OPENCLASH_CORE_CONFIG]

```
待填入：
- 运行模式：Fake-IP（TUN）
- Fake-IP 范围：198.0.0.0/8
- DNS 劫持：启用，端口 7874
- SOCKS5：启用，端口 7893
- 本地 DNS 劫持：启用
```

1.2 内核 IP 转发

[SYSCTL_IP_FORWARD]

```
待填入：
- 临时开启：echo 1 > /proc/sys/net/ipv4/ip_forward
- 永久写入：echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
- 验证：sysctl net.ipv4.ip_forward
```

1.3 确认 TUN 设备

[TUN_VERIFY]

```
待填入：
- 查看命令：ip a | grep utun
- 端口监听：ss -tlnp | grep -E '7874|7893'
```

---

二、Ubuntu 端（192.168.9.3）

2.1 安装 mini-ppdns

[MINI_PPDNS_INSTALL]

```
待填入：
- 下载：wget -q [release_url] -O /usr/local/bin/mini-ppdns
- 赋权：chmod +x /usr/local/bin/mini-ppdns
```

2.2 配置文件 `/etc/mini-ppdns.ini`

```ini
[dns]
# [PRIMARY_DNS]
# 待填入：192.168.9.2:7874

[fall]
# [FALLBACK_DNS]
# 待填入：
# 223.5.5.5:53
# 119.29.29.29:53

[listen]
# [LISTEN_ADDR]
# 待填入：0.0.0.0:53

[adv]
# [ADVANCED_PARAMS]
# 待填入：
# qtime = 250
# aaaa = no
# lite = yes
# boguspriv = 1
```

2.3 写入 systemd 自启

[SYSTEMD_SERVICE]

```
待填入：
- 文件路径：/etc/systemd/system/mini-ppdns.service
- [Unit] After=network-online.target
- [Service] ExecStart=/usr/local/bin/mini-ppdns -config /etc/mini-ppdns.ini -d
- [Install] WantedBy=multi-user.target
- 重载：systemctl daemon-reload
- 启停：systemctl enable --now mini-ppdns
```

2.4 本地验证

[MINI_PPDNS_CHECK]

```
待填入：
- 状态：systemctl status mini-ppdns
- 查端口：ss -tlnp | grep :53
- 测 FakeIP：dig @192.168.9.3 www.google.com（应返回 198.0.0.x）
- 测国内：dig @192.168.9.3 www.baidu.com（应返回真实 IP）
```

---

三、iKuai 端（192.168.9.1）

3.1 DHCP 服务

[IKUAI_DHCP_SETUP]

```
待填入：
- 路径：网络设置 → DHCP 设置 → DHCP 服务端 → 编辑 lan1
- 首选 DNS：192.168.9.3
- 备选 DNS：223.5.5.5（可选）
- 网关：192.168.9.1（保持）
- 租期：86400
- 操作：保存 → 应用 → 重启 DHCP 服务
```

3.2 静态路由（FakeIP 引流）

[IKUAI_STATIC_ROUTE]

```
待填入：
- 路径：网络设置 → 路由设置 → 静态路由 → 添加
- 目的地址：198.0.0.0
- 子网掩码：255.0.0.0（/8）
- 网关：192.168.9.2
- 接口：LAN1
- 优先级：1
- 描述：OpenClash-FakeIP
- 操作：保存 → 应用
- 验：路由表搜 198.0.0.0，状态为已启用
```

3.3 DHCP Option 121（自动下发路由）

[IKUAI_OPTION121]

```
待填入：
- 路径：DHCP 服务端编辑 → 高级设置 → 自定义选项
- Option：121
- 类型：字符串
- 值：198.0.0.0/8 192.168.9.2
- 备选 Hex：08 C6 00 00 00 00 00 00 00 02
- 操作：保存 → 应用
```

3.4 NAT 过滤（保源 IP）

[IKUAI_NAT_FILTER]

```
待填入：
- 路径：网络设置 → NAT 规则 / 防火墙 → NAT 过滤 → 添加
- 源地址：192.168.9.0/24
- 目的地址：198.0.0.0/8
- 动作：不 NAT / 过滤
- 方向：转发
- 描述：FakeIP-No-NAT
- 关键：保存后拖至规则列表最顶部
```

3.5 ACL 放行（如有阻断）

[IKUAI_ACL_ALLOW]

```
待填入：
- 路径：网络安全 → ACL 规则
- 规则1：源 LAN → 目的 192.168.9.2，动作允许
- 规则2：源 LAN → 目的 192.168.9.3，端口 53，动作允许
- 置于所有拒绝规则之前
```

---

四、重启与验证

4.1 重启次序

[REBOOT_SEQUENCE]

```
待填入：
1. Ubuntu：systemctl restart mini-ppdns
2. OpenWrt：重启 OpenClash 服务
3. iKuai：重启网络服务或整机
4. 客户端：ipconfig /release && ipconfig /renew
```

4.2 验证清单

验证项	命令	预期结果	
DNS 发 FakeIP	`nslookup www.google.com 192.168.9.3`	`198.0.0.x`	
国内 DNS 正常	`nslookup www.baidu.com 192.168.9.3`	真实国内 IP	
路由指向正确	`tracert -d 198.0.0.1`	第一跳 `192.168.9.2`	
TUN 存在	`ip a \| grep utun`	显示 utun 接口	
源 IP 不堕	OpenClash 连接日志	显示客户端真实 IP，非 `192.168.9.1`	
灾备生效	`docker stop paopaodns`（如有）后测百度	仍可解析	

[VERIFY_DETAIL_COMMANDS]

```
待填入：各验证命令的完整输出样例
```
