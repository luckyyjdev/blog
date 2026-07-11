+++
title = "Quest 3 能看 YouTube 4K 却下不了应用？—— 一次排查网络诊断记录"
date = 2026-07-11
draft = false
tags = [ "网络", "openwrt", "openclash", "quest" ]
categories = [ "技术" ]
summary = ""
+++

# Quest 3 能看 YouTube 4K 却下不了应用？——  一次排查网络诊断记录

> 一台 Quest 3，科学上网环境（OpenWrt 旁路由 + OpenClash），YouTube 4K 稳如老狗，应用商店下载却只有几 KB/s 然后彻底卡死。这篇文章记录了我从 ADB 上手到定位根因的完整排查过程——中间绕了五个弯，每一次「以为搞定了」都被现实打脸，最后发现是一个旁路由架构下极具迷惑性的 IPv6 坑。

## 背景

网络拓扑长这样：

```
互联网
  │
红米主路由 (192.168.28.1)  ← WiFi AP，Quest 3 连它
  │   DHCP 下发：网关=192.168.28.2，DNS=192.168.28.2 + 114.114.114.114
  │
OpenWrt 旁路由 (192.168.28.2)  ← 跑 OpenClash 做代理
```

这是国内很常见的「旁路由」方案：主路由负责 WiFi 和 DHCP，OpenWrt 旁路由专门跑代理，客户端把网关指向旁路由就能走代理。

**症状**：Quest 3 上 YouTube 能流畅看 4K，但应用商店下载任何应用都是几 KB/s 然后完全不动。

这个症状很有意思——能看 4K 说明带宽和代理本身没问题，那为什么下载不行？带着这个疑问开始排查。

---

## 第一轮：ADB 上手，DNS 污染现形

Quest 3 连上电脑开了 ADB。先看网络配置：

```bash
$ adb shell ip addr show wlan0
inet 192.168.28.7/24 ...   # IP 正常
$ adb shell dumpsys connectivity | grep Dns
DnsAddresses: [ /fe80::46f7:70ff:febd:c40%wlan0, /192.168.28.2, /114.114.114.114 ]
```

DNS 有三个：一个 IPv6 链路本地地址，OpenWrt（192.168.28.2），还有 114.114.114.114。先记下，继续查。

用 `ping` 看 Meta 关键域名解析到什么 IP（Quest 上没有 nslookup，但 ping 会显示解析结果）：

```
graph.oculus.com       → 104.244.42.197   ← 这是 Twitter 的 IP！
securecdn.oculus.com   → 157.240.253.55   ← 正确的 Meta IP
www.oculus.com         → 69.171.235.22    ← 正确的 Meta IP
scontent.fbcdn.net     → unknown host     ← 解析失败
download.oculus.com    → unknown host     ← 解析失败
graphene.oculus.com    → unknown host     ← 解析失败
```

`graph.oculus.com`（Quest 商店的核心 API）解析到了 **104.244.42.197**——这是 Twitter 的 IP 段，根本不是 Meta 的。这是典型的 **GFW DNS 污染**：对被墙域名返回一个错误的 IP。

再看实时连接，问题更明显：

```bash
$ adb shell toybox netstat -tn | grep ESTABLISHED
tcp  192.168.28.7:xxx  104.244.42.197:443  TIME_WAIT
tcp  192.168.28.7:xxx  104.244.42.197:443  TIME_WAIT
tcp  192.168.28.7:xxx  104.244.42.197:443  TIME_WAIT
... (几十个，全是连这个错误 IP 的)
```

Quest 3 在疯狂连接这个错误的 Twitter IP，TLS 握手失败（证书对不上），断开重试，再失败……死循环。这就是下载卡死的直接原因。

**为什么 YouTube 没事？** 因为 `googlevideo.com` 在 OpenClash 的规则列表里，走代理 DNS 解析（干净）；而 Meta 的 CDN 域名没被规则覆盖，走直连 DNS，被 GFW 污染。

### 第一次修复

1. OpenClash 添加 Meta 域名代理规则：`oculus.com`、`fbcdn.net`、`facebook.com`、`meta.com` 等
2. 红米路由 DHCP 移除 `114.114.114.114`，DNS 只留 `192.168.28.2`

改完重启 Quest WiFi，满心欢喜地验证——

---

## 第二轮：规则加了，还是污染

重新测试，`graph.oculus.com` 还是解析到污染 IP。但有个细节引起了注意：用电脑直接查 OpenWrt 的 DNS，结果却是对的：

```bash
# 电脑上直查 OpenWrt DNS
$ nslookup graph.oculus.com 192.168.28.2
Address: 57.144.45.141   ← 正确的 Meta IP！
```

同一个 DNS 服务器（192.168.28.2），电脑查到正确 IP，Quest 查到污染 IP。这不对劲。

深入想，电脑查到的是**真实 IP**（57.144.45.141），不是 fake-ip 段的 `198.18.x.x`。这说明 **OpenClash 跑的是 Redir-Host 模式，不是 Fake-IP 模式**。

> **Redir-Host vs Fake-IP**：
> - Redir-Host 模式：DNS 在本地解析出真实 IP 返回给客户端，再根据 IP/域名规则决定是否代理。问题在于「本地解析真实 IP」这一步可能被 GFW 污染。
> - Fake-IP 模式：匹配规则的域名直接返回一个假 IP（198.18.x.x），真实解析在代理节点完成，彻底规避 DNS 污染。

Redir-Host 模式下，即使规则加对了，DNS 解析过程本身仍可能被污染。必须切到 Fake-IP。

### 第二次修复

OpenClash 全局设置里，运行模式从 Redir-Host 改为 **Fake-IP（增强）模式**，并检查 fake-ip-filter 列表不含 Meta 域名。

改完再用电脑验证：

```bash
$ nslookup graph.oculus.com 192.168.28.2
Address: 198.18.0.17   ← fake-ip！对了！
```

电脑端 fake-ip 完全正常。这次总该行了吧——

---

## 第三轮：fake-ip 生效了，Quest 却还异常

重启 Quest WiFi，再测：

```
graph.oculus.com     → 69.171.235.22    ← 不是 fake-ip！
scontent.fbcdn.net   → unknown host     ← 还是失败
graphene.oculus.com  → unknown host     ← 还是失败
```

电脑查全对，Quest 查全错。同一个 DNS 服务器，为什么结果不同？

回头再看 Quest 的 DNS 配置：

```
DnsAddresses: [ /fe80::46f7:70ff:febd:c40%wlan0, /192.168.28.2 ]
```

**IPv6 链路本地地址 `fe80::46f7:70ff:febd:c40` 排在第一位**。Android 的 DNS 解析器会优先用排在前面的 DNS 服务器。也就是说，Quest 的 DNS 查询优先发给了这个 IPv6 地址，而不是 IPv4 的 192.168.28.2。

而 OpenClash 的 fake-ip 只接管了 IPv4 DNS。IPv6 DNS 查询绕过了 OpenClash，直连出去被 GFW 污染。

证据：连接里同时存在 `198.18.0.17`（fake-ip，走代理✅）和 `174.132.167.252`（污染✗），取决于这次查询走了 IPv4 还是 IPv6。

**根因似乎找到了**：Quest 优先用 IPv6 DNS，绕过了 fake-ip。那去掉这个 IPv6 DNS 不就行了？

---

## 第四轮：禁了 IPv6，fe80 却阴魂不散

先在 OpenWrt 上禁用 IPv6（RA 服务、DHCPv6、甚至 `sysctl net.ipv6.conf.all.disable_ipv6=1`），重启 Quest WiFi——

```
DnsAddresses: [ /fe80::46f7:70ff:febd:c40%wlan0, /192.168.28.2 ]
```

`fe80::` 还在！OpenWrt 的 IPv6 都禁了，这个地址哪来的？

这时候一个关键操作改变了排查方向——**用 MAC 地址定位 fe80 的真正归属**。

IPv6 链路本地地址 `fe80::` 的后半段是基于网卡 MAC 地址生成的（EUI-64）。把 `fe80::46f7:70ff:febd:c40` 反推 MAC：

```
fe80::46f7:70ff:febd:c40
         ↓ EUI-64 反推（中间插入 ff:fe，第一字节翻转第7位）
MAC = 44:f7:70:bd:0c:40
```

然后查局域网里两个路由器的 MAC：

```bash
$ arp -a | grep "192.168.28.[12]"
192.168.28.1    44-f7-70-bd-0c-40    ← 红米主路由！匹配！
192.168.28.2    52-54-00-f8-c6-ff    ← OpenWrt，不匹配
```

**这个 fe80:: 是红米主路由的，不是 OpenWrt 的！**

前面在 OpenWrt 上怎么折腾都没用，因为发 IPv6 RA（路由通告）的根本不是 OpenWrt，是红米。红米通过 RA 把自己的 fe80 地址作为 IPv6 DNS 通告给局域网所有设备，Quest 收到后优先使用，绕过了 OpenWrt 的 fake-ip。

> **这里有个认知误区**：我一直以为「Quest 连到旁路由」就意味着所有网络信息都来自 OpenWrt。实际上 Quest 的 WiFi 物理连的是红米（AP），红米、OpenWrt、Quest 在同一个二层网段。IPv6 RA 是二层组播（ff02::1），红米发的 RA 直接广播到同网段所有设备，不经过网关。OpenWrt 作为旁路由根本拦不住。

### 方案探索

想让 Quest 不用红米的 IPv6 DNS，有几条路：

1. **红米 IPv6 DNS 改填 OpenWrt 的 fe80**——但这样其他设备也会拿到 fake-ip，影响面大
2. **关闭红米 IPv6 总开关**——简单彻底，但全网没 IPv6
3. **设备端禁 IPv6**——Quest 没权限，做不到

权衡后选择了方案 2：关红米 IPv6 总开关。IPv4 上网和代理完全不受影响，只是没了 IPv6 而已，对日常使用几乎无感。

---

## 第五轮：关红米 IPv6，彻底解决

红米管理界面关闭 IPv6 总开关，重启 Quest WiFi，验证：

```
DnsAddresses: [ /192.168.28.2 ]        ← fe80:: 消失了！只剩 IPv4
```

```
graph.oculus.com     → 198.18.0.7     ✅ fake-ip
scontent.fbcdn.net   → 198.18.0.17    ✅（此前一直解析失败）
graphene.oculus.com  → 198.18.0.18    ✅（此前一直解析失败）
securecdn.oculus.com → 198.18.0.5     ✅
youtube.com          → 198.18.0.21    ✅
www.oculus.com       → 198.18.0.22    ✅
```

全部 fake-ip 生效！再看连接：

```bash
$ adb shell toybox netstat -tn | grep ESTABLISHED
tcp  192.168.28.7:xxx  198.18.0.7:443   ESTABLISHED   ← 商店 API 正常
tcp  192.168.28.7:xxx  198.18.0.8:443   ESTABLISHED
tcp  192.168.28.7:xxx  198.18.0.19:443  ESTABLISHED
... (全是 198.18.x.x，零污染 IP)
```

打开 Quest 商店，下载飞快，问题彻底解决。

---

## 原理深挖：为什么旁路由挡不住主路由的 RA

这是整个排查里最值得讲的一个点，也是旁路由架构最容易踩的坑。

### 三层 vs 二层

「连到旁路由」实际上只改了两样东西：
- **网关** → OpenWrt（192.168.28.2）
- **IPv4 DNS** → OpenWrt（192.168.28.2）

但 WiFi 的**物理接入**还是红米（SSID 是红米的）。Quest、红米、OpenWrt 三者处在同一个二层网段。

**网关是三层概念**，控制的是「IP 包往哪转发」。**IPv6 RA 是二层组播**（目标地址 ff02::1，所有节点），红米定期在局域网里广播，同网段所有设备直接收到，根本不经过网关。

打个比方：你跟私人秘书说「以后我的信都你来转交」（设网关=OpenWrt），但小区广播站还是对着整个小区大喇叭喊（RA 二层组播），你在家直接听到了广播，秘书压根没经手。要让广播不影响你，要么让广播站闭嘴（关红米 IPv6），要么戴耳塞（设备端禁 IPv6）。

### Android 的「加成」行为

除了 RA 通告的 RDNSS（DNS 服务器选项），Android 还有一个行为：把 **IPv6 默认网关的链路本地地址自动加到 DNS 列表**。红米的 RA 告诉 Quest「IPv6 默认网关是我」，Android 顺势把红米的 fe80 也塞进了 DNS，而且排在 IPv4 DNS 前面优先使用。

这就是为什么即使红米的 RA 不带 RDNSS 选项，只要还在发 RA（通告自己是 IPv6 网关），Quest 就会拿到红米的 fe80 当 DNS。在 OpenWrt 上禁 IPv6 没用，因为发 RA 的是红米。

### 为什么在 OpenWrt 上折腾无效

排查中我在 OpenWrt 上做了这些，全部无效：
- 禁用 RA 服务
- 禁用 DHCPv6 服务
- `sysctl net.ipv6.conf.all.disable_ipv6=1` 彻底关 IPv6 协议栈

因为这些操作只影响 OpenWrt 自己发不发 RA，不影响红米。红米的 RA 照样二层组播到 Quest。

---

## 排查方法论总结

这次排查用了几个很有效的诊断手法，值得记下来：

### 1. 电脑直查旁路由 DNS vs 设备解析结果对比

这是定位「OpenClash 配置问题」还是「DNS 路径绕过」的关键。

```bash
# 电脑直查 OpenWrt DNS
nslookup graph.oculus.com 192.168.28.2
# 设备上解析
adb shell ping -c1 graph.oculus.com
```

- 两者结果一致 → OpenClash 配置有问题
- 电脑对、设备错 → 设备的 DNS 查询没走 OpenClash（被抢答/绕过/缓存）

### 2. 用 MAC 地址定位 fe80 归属

IPv6 链路本地地址 `fe80::` 后半段基于 MAC 生成（EUI-64）。反推公式：

```
fe80::XXYY:ZZff:feWW:VVUU  →  MAC = (XX XOR 02):YY:ZZ:WW:VV:UU
```

再对照 ARP 表里各设备的 MAC，就能确定 fe80 到底是谁的。这次正是靠这一招识破了「fe80 来自红米而非 OpenWrt」。

### 3. netstat 看实际连接目标

光看 DNS 解析不够，`netstat` 能看到应用实际连到了哪个 IP。如果连接里全是某个污染 IP 且都是 TIME_WAIT，说明 TLS 握手失败死循环，是域名被污染的铁证。

### 4. 判断 OpenClash 运行模式

解析一个走代理的域名：
- 返回真实 IP（如 157.240.x.x）→ Redir-Host 模式
- 返回 198.18.x.x → Fake-IP 模式

Redir-Host 模式天然有 DNS 泄漏/污染风险，代理场景下建议用 Fake-IP。

### 5. ADB 重启 WiFi 获取新 DHCP 配置

```bash
adb shell cmd wifi set-wifi-enabled disabled
adb shell cmd wifi set-wifi-enabled enabled
```

改完路由器 DHCP 后，用这个让设备重新获取配置，不用手动在头显里操作。

---

## 最终生效的完整修复

按顺序四步，缺一不可：

| 步骤 | 操作 | 解决的问题 |
|------|------|-----------|
| 1 | OpenClash 添加 oculus.com/fbcdn.net/facebook.com/meta.com 等域名代理规则 | Meta 域名走直连被污染 |
| 2 | OpenClash 切到 Fake-IP（增强）模式 | Redir-Host 模式 DNS 仍被污染 |
| 3 | 红米 DHCP 移除 114.114.114.114 | 备用 DNS 抢答返回污染结果 |
| 4 | 关闭红米路由 IPv6 总开关 | 红米 RA 通告的 fe80 IPv6 DNS 绕过 fake-ip |

前 3 步是「让 OpenClash fake-ip 正确工作」，第 4 步是「让设备真正用上 fake-ip 而不被 IPv6 DNS 绕过」。第 4 步是最后一个、也是最隐蔽的坑。

---

## 写在最后

这次排查最大的教训：**旁路由架构下，主路由的 IPv6 RA 是二层广播，旁路由拦不住。** 如果主路由还开着 IPv6，客户端会拿到主路由的 IPv6 DNS，优先用它查询，直接绕过旁路由上的 OpenClash fake-ip。这个问题在「能看 YouTube 但下载不了」这种部分功能异常的场景下特别隐蔽，因为 IPv4 代理是好的，只有走 IPv6 DNS 的查询会出问题。

如果你也在用旁路由 + OpenClash，遇到代理「时好时坏」「部分域名不行」的问题，不妨查一下设备 DNS 列表里有没有一个排在前面的 `fe80::` 地址——如果有，很可能就是这个坑。

希望这篇记录能帮到踩同样坑的人。
