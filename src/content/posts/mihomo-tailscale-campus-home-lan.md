---
title: 用 mihomo 内置 Tailscale 在校园网访问家里内网
published: 2026-05-23
tags: [VPS,工具,教程]
category: 教程
draft: false
---

# 用 mihomo 内置 Tailscale 在校园网访问家里内网


> 写于 2026-05-23。本文记录一次从校园网访问家里内网的实际折腾过程：家里用 Debian 跑 Tailscale subnet router，手机用 Android root + Box for Root，电脑用 Windows + Clash Party，最终实现不开 Android 官方 Tailscale App，也能直接访问家里的 `192.168.1.0/24` 和 `192.168.0.0/24`。

## 背景

我原来的需求很简单：人在学校，想访问家里的内网服务。

家里有软路由、光猫、NAS、虚拟机、LuCI、SSH 等一堆服务，如果用 frp 一个个穿透端口，会很麻烦：端口多，映射难记，后期维护也不舒服。

Tailscale 很适合解决这个问题。家里只要有一台设备作为 subnet router，把家里的内网网段发布到 tailnet，外面的设备就可以像在家里一样访问内网 IP。

但是 Android 上有个现实问题：官方 Tailscale App 会占用 Android 的 VPNService。对于已经 root 并且使用 Box for Root、Nikki、Clash Meta 之类透明代理模块的设备来说，Tailscale App 一开，系统 VPN 路由就会和代理模块打架。结果就是 Tailscale 能用了，但原本的代理分流可能失效。

这次的核心思路是：

```text
手机或电脑上的普通流量 -> 继续走 mihomo 原来的代理分流
访问家里内网的流量 -> 走 mihomo 内置 Tailscale 出站
```

mihomo 从 `v1.19.25` 开始正式加入了 Tailscale outbound support，配置里可以直接写 `type: tailscale`。参考：

- [mihomo v1.19.25 Release](https://github.com/MetaCubeX/mihomo/releases/tag/v1.19.25)
- [mihomo Tailscale 出站文档](https://wiki.metacubex.one/config/proxies/tailscale/)

## 我的网络拓扑

我的环境大概是这样：

```text
校园网 NAT4
  |
  |-- Android 手机
  |     |-- root
  |     |-- Box for Root
  |     |-- mihomo v1.19.25+
  |
  |-- Windows 电脑
        |-- Clash Party
        |-- mihomo v1.19.25+

公网
  |
  |-- 南京腾讯云
        |-- 自建 DERP 中继

家里
  |
  |-- 光猫: 192.168.0.0/24
  |-- 软路由 ImmortalWrt: 192.168.1.0/24
  |-- Nikki 透明代理
  |-- Debian 虚拟机
        |-- 不走 Nikki
        |-- 运行 Tailscale
        |-- 作为 subnet router 发布:
              192.168.1.0/24
              192.168.0.0/24
```

最终效果是：

```text
Android App / Windows 软件
  -> mihomo
  -> Home LAN 策略组
  -> Tailscale-Home 出站
  -> 家里 Debian subnet router
  -> 家里内网设备
```

## 一些前提

本文假设你已经有一个 Tailscale 账号，并且可以进入 Tailscale Admin Console。

需要准备：

- 家里一台 Debian 机器，虚拟机、物理机都可以
- Debian 能访问家里的内网，比如 `192.168.1.0/24`、`192.168.0.0/24`
- Android 手机已经 root，并运行 Box for Root
- Windows 电脑运行 Clash Party
- mihomo 内核版本为 `v1.19.25` 或更新版本

如果启动时报：

```text
proxy 0: unsupport proxy type: tailscale
```

说明你的 mihomo 内核太旧，不支持 `type: tailscale`。需要升级到 `v1.19.25` 或更新版本。

另外，家里作为 subnet router 的 Debian 最好走直连，不要被软路由上的 Nikki、透明代理或分流规则二次代理。原因很简单：这台 Debian 是整套链路的“家里入口”，它需要稳定连接 Tailscale 控制面、DERP、STUN 和其它 tailnet 节点。如果它的 Tailscale 外层连接再被家里代理套一层，排错会变复杂，甚至可能出现绕路或回环。

如果你的 Debian 必须经过软路由出网，建议至少在软路由上给这台 Debian 做直连策略。

## 第一部分：在家里 Debian 部署 Tailscale subnet router

### 1. 安装 Tailscale

在 Debian 上执行：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

官方 Linux 安装文档也推荐这个安装脚本。参考：[Install Tailscale on Linux](https://tailscale.com/docs/install/linux)。

安装完成后，先登录 Tailscale：

```bash
sudo tailscale up --hostname=home-debian-router
```

命令会输出一个登录链接。打开链接，登录你的 Tailscale 账号，授权这台 Debian 加入 tailnet。

确认状态：

```bash
tailscale status
tailscale ip -4
```

如果能看到 `100.x.y.z` 这样的 Tailscale IP，说明 Debian 已经入网。

### 2. 开启 Linux IP forwarding

Subnet router 的本质是让 Debian 帮其它 tailnet 设备转发到家里内网，所以 Debian 必须开启 IP 转发。

执行：

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

检查：

```bash
sysctl net.ipv4.ip_forward
```

应该看到：

```text
net.ipv4.ip_forward = 1
```

Tailscale 官方 subnet router 文档也明确要求 Linux subnet router 开启 IP forwarding。参考：[Tailscale subnet routers](https://tailscale.com/docs/features/subnet-routers)。

### 3. 发布家里内网路由

我的家里有两个网段：

```text
192.168.1.0/24  软路由后面的主 LAN
192.168.0.0/24  光猫网段
```

在 Debian 上发布这两个网段：

```bash
sudo tailscale set --advertise-routes=192.168.1.0/24,192.168.0.0/24
```

这一步只是“声明我要发布这些网段”，还需要去 Tailscale 后台批准。

### 4. 在 Tailscale 后台批准 subnet routes

打开：

```text
https://login.tailscale.com/admin/machines
```

找到刚才的 Debian 设备，比如：

```text
home-debian-router
```

进入它的设置，在 Subnet routes 里启用：

```text
192.168.1.0/24
192.168.0.0/24
```

保存。

如果这是长期在线的家里网关，建议顺手给它关掉 key expiry。否则某天设备 key 过期，subnet route 会保留但变得不可达，排查起来很绕。

### 5. 关于 SNAT

Tailscale subnet router 默认会做 SNAT，也就是家里内网设备看到的来源是 Debian subnet router，而不是远端设备的真实 Tailscale IP。

默认 SNAT 的好处是最省心：家里的光猫、软路由、NAS 都不需要额外配置回程路由。

如果你关闭 SNAT：

那家里内网设备要知道如何回到 Tailscale 网段，也就是要有类似这样的回程路由：

```text
100.64.0.0/10 via Debian 的家里内网 IP
```

普通家庭网络建议保持默认 SNAT，不要一开始就把复杂度拉满。只有当你明确需要让家里内网设备看到远端 Tailscale 设备的真实 `100.x` 地址时，再按官方文档调整 `--snat-subnet-routes=false` 和回程路由。

## 第二部分：生成 mihomo 用的 Tailscale auth key

mihomo 的 `type: tailscale` 会把 mihomo 自己变成一个 Tailscale 节点。它不能复用你手机或电脑上已有的官方 Tailscale App 身份，而是会在后台显示成一个新设备，比如：

```text
android-box
win-mihomo
```

第一次加入 tailnet 需要一个 auth key。

打开：

```text
https://login.tailscale.com/admin/settings/keys
```

点击 Generate auth key，建议这样选：

```text
Description: android-box-mihomo 或 win-mihomo
Reusable: 关闭
Expiration: 1 day
Ephemeral: 关闭
Tags: 关闭
Pre-approved: 如果有这个选项，可以打开
```

生成后会得到：

```text
tskey-auth-xxxxxxxxxxxxxxxx
```

注意不要用 `tskey-api-`，要用 `tskey-auth-`。

这个 key 只用于第一次登录。只要 `state-dir` 保留下来，后面重启不需要重新生成 key。auth key 和 `state-dir` 都很敏感，不要发到公开仓库。

参考：[Tailscale auth keys](https://tailscale.com/kb/1085/auth-keys)。

## 第三部分：Android + Box for Root 配置

Android 上的问题是官方 Tailscale App 会占用 VPNService，所以这里不要开 Android 官方 Tailscale App。

我们让 Box for Root 里的 mihomo 自己作为 Tailscale 节点入网。

### 1. 新增 Tailscale 出站

在 mihomo 配置的 `proxies:` 里新增：

```yaml
proxies:
  - name: Tailscale-Home
    type: tailscale
    hostname: android-box
    auth-key: "tskey-auth-REPLACE_ME"
    control-url: https://controlplane.tailscale.com
    state-dir: /data/adb/box/mihomo/tailscale
    ephemeral: false
    udp: true
    accept-routes: true
    ip-version: ipv4-prefer
```

把 `tskey-auth-REPLACE_ME` 换成刚刚生成的 auth key。

`state-dir` 很重要，它保存的是这个 Tailscale 节点的身份。不要删，也不要复制给其它设备。

### 2. 新增 Home LAN 策略组

加一个专门控制家里内网的策略组：

```yaml
proxy-groups:
  - name: Home LAN
    type: select
    proxies:
      - Tailscale-Home
      - DIRECT
      - Proxy
```

这样做的好处是可以在面板里手动切换：

```text
Tailscale-Home  在学校时使用，访问家里内网
DIRECT          在家里 Wi-Fi 下使用，直接访问内网
Proxy           临时排错用，一般用不到
```

如果你的 `Select` 组用了 `include-all: true`，建议排除 `Tailscale-Home`，避免它混进普通代理节点：

```yaml
exclude-filter: "^(Tailscale-Home)$"
```

### 3. 规则必须放在 private_ip 前面

如果你的配置里有：

```yaml
- RULE-SET,private_ip,DIRECT,no-resolve
```

那这条会提前把 `192.168.x.x` 送去直连。为了让家里网段走 Tailscale，必须把规则放在它前面：

```yaml
rules:
  - IP-CIDR,192.168.1.0/24,Home LAN,no-resolve
  - IP-CIDR,192.168.0.0/24,Home LAN,no-resolve
  - IP-CIDR,100.64.0.0/10,Home LAN,no-resolve
  - RULE-SET,private_ip,DIRECT,no-resolve
```

`100.64.0.0/10` 是 Tailscale 使用的 CGNAT 网段。加上它后，可以直接访问 tailnet 里其它设备的 `100.x.y.z` 地址。

### 4. 重启 Box for Root

可以在手机终端里执行：

```bash
su
/data/adb/box/scripts/box.service restart
/data/adb/box/scripts/box.iptables renew
```

启动后去 Tailscale 后台看 Machines 页面，应该能看到：

```text
android-box
```

再测试访问：

```text
http://192.168.1.1
http://192.168.0.1
```

在 mihomo 面板里看连接，目标是 `192.168.x.x` 时应该命中：

```text
Home LAN -> Tailscale-Home
```

## 第四部分：Windows + Clash Party 配置

Windows 上有两种玩法：

```text
简单玩法：不用 TUN，浏览器或软件手动走 127.0.0.1:7890/7891
舒服玩法：开 TUN，但只接管家里网段
```

我最终采用第二种。

### 1. 更新 Clash Party 的 mihomo 内核

Windows 普通 Intel/AMD 电脑建议下载：

```text
mihomo-windows-amd64-compatible-v1.19.25.zip
```

如果是 ARM Windows，才选：

```text
mihomo-windows-arm64-v1.19.25.zip
```

解压后把里面的 exe 放到 Clash Party 的内核目录。

如果替换稳定内核，命名为：

```text
mihomo.exe
```

如果替换 Alpha 内核槽位，命名为：

```text
mihomo-alpha.exe
```

然后在 Clash Party 里选择对应内核。

### 2. Windows 使用独立 Tailscale 设备名

不要把手机的 `hostname`、`auth-key`、`state-dir` 原样复制给电脑。

Windows 上建议：

```yaml
proxies:
  - name: Tailscale-Home
    type: tailscale
    hostname: win-mihomo
    auth-key: "tskey-auth-REPLACE_ME"
    control-url: https://controlplane.tailscale.com
    state-dir: ./tailscale-state
    ephemeral: false
    udp: true
    accept-routes: true
    ip-version: ipv4-prefer
```

每台设备都应该是独立身份：

```text
手机: android-box
电脑: win-mihomo
```

每台设备第一次启动都生成一个新的 auth key。

### 3. Windows 系统代理绕过列表的坑

Windows 代理绕过列表里经常会有：

```text
localhost;127.*;192.168.*;10.*;172.16.*;172.17.*;...
```

如果保留 `192.168.*`，浏览器访问 `192.168.1.1` 时不会走 Clash Party，自然也不会走 `Tailscale-Home`。

如果只是浏览器访问，可以删掉 `192.168.*`。

但是 FinalShell、SSH、某些游戏启动器、某些客户端并不一定遵守系统 HTTP 代理。它们直接连 `192.168.1.x`，流量根本没进 mihomo。

所以更好的办法是开 TUN。

### 4. 重点：Windows TUN 只接管家里网段

一开始我开了全局 TUN，结果正常代理都坏了，日志里出现类似：

```text
reject loopback connection to 223.5.5.5:53
reject loopback connection to 119.29.29.29:53
```

原因是全局 TUN 把 DNS、普通代理流量，甚至 Tailscale 自己的外层连接都抓进来了，容易形成回环。

正确思路不是让 TUN 接管所有流量，而是只接管家里网段：

```yaml
tun:
  enable: true
  stack: mixed
  device: Mihomo
  auto-route: true
  auto-detect-interface: true
  strict-route: false
  route-address:
    - 192.168.1.0/24
    - 192.168.0.0/24
    - 100.64.0.0/10
```

不要为了这个场景开这些：

```yaml
dns-hijack:
auto-redirect:
strict-route: true
```

mihomo TUN 文档里说明，`auto-route` 会自动设置路由，`route-address` 可以在启用 `auto-route` 时自定义路由网段；`strict-route` 在 Windows 上会加防火墙规则，某些场景下可能影响应用。参考：[mihomo TUN 文档](https://wiki.metacubex.one/config/inbound/tun/)。

最终路径是：

```text
FinalShell / 浏览器 / 任意软件
  -> Windows 路由表发现目标是 192.168.1.0/24
  -> 送进 Mihomo TUN
  -> mihomo 规则命中 Home LAN
  -> Tailscale-Home
  -> 家里 Debian subnet router
  -> 家里内网设备
```

普通网站、普通 DNS、普通代理流量不进这条 TUN 路由，所以原来的代理分流不会被破坏。

### 5. Windows 测试

如果没有开 TUN，可以强制走 HTTP 代理测试：

```powershell
curl.exe -v -x http://127.0.0.1:7890 http://192.168.1.1/ --connect-timeout 5
```

如果开了只接管家里网段的 TUN，直接测试：

```powershell
curl.exe -v http://192.168.1.1/ --connect-timeout 5
```

成功时会看到类似：

```text
HTTP/1.1 200 OK
LuCI - Lua Configuration Interface
```

这时 FinalShell 也可以直接填：

```text
Host: 192.168.1.x
Port: 22
```

不需要再给每个 SSH 连接单独指定 SOCKS5。

如果不开 TUN，FinalShell 里可以手动指定：

```text
SOCKS5
Host: 127.0.0.1
Port: 7891
```

## 第五部分：验证 Tailscale 是否直连

在家里 Debian 上可以 ping 手机或电脑的 Tailscale 节点：

```bash
tailscale ping android-box
tailscale ping win-mihomo
```

如果看到类似：

```text
pong from win-mihomo (100.110.85.59) via 120.xxx.xxx.xxx:4298 in 26ms
```

这说明已经走 UDP 直连。`120.xxx.xxx.xxx` 是校园网出口 IP。

如果走 DERP，通常会显示 DERP 或 relay 相关字样。

持续 ping 可以用：

```bash
tailscale ping --c=0 --until-direct=false win-mihomo
```

如果你的版本不支持 `--c=0` 持续 ping，可以用：

```bash
while true; do
  tailscale ping --c=1 --timeout=3s --until-direct=false win-mihomo
  sleep 1
done
```

手机延迟可能比电脑更抖，这是正常的。Android 的 Wi-Fi 省电、后台调度、root 透明代理和 tsnet 用户态栈都会带来一些波动。只要不是频繁掉 DERP 或超时，访问网页、SSH、LuCI、NAS 通常都够用。

## 常见问题

### 1. 报 `unsupport proxy type: tailscale`

内核太旧。升级到 mihomo `v1.19.25` 或更新版本。

### 2. Tailscale 后台看到新设备，但访问家里 IP 不通

按顺序检查：

```text
1. 家里 Debian 是否发布了 192.168.1.0/24 和 192.168.0.0/24
2. Tailscale 后台是否批准了这两个 subnet routes
3. mihomo 规则里家里网段是否在 private_ip,DIRECT 前面
4. Home LAN 策略组是否选中了 Tailscale-Home
5. Windows 是否被代理绕过列表拦住
6. 是否开了全局 TUN 导致 DNS 或外层连接回环
```

### 3. Windows 上 `curl -x 127.0.0.1:7893` 连接拒绝

说明 Clash Party 实际没有监听 `7893`。

检查端口：

```powershell
netstat -ano | findstr "7890 7891 7892 7893 7894 9090"
```

很多 Clash Party 配置里实际 HTTP/Mixed 端口是 `7890`，SOCKS 是 `7891`。

### 4. 开 TUN 后普通代理也坏了

多半是开了全局 TUN、`dns-hijack` 或 `strict-route: true`，导致 DNS/普通代理流量回环。

针对这个场景，Windows 上建议只写：

```yaml
tun:
  enable: true
  stack: mixed
  device: Mihomo
  auto-route: true
  auto-detect-interface: true
  strict-route: false
  route-address:
    - 192.168.1.0/24
    - 192.168.0.0/24
    - 100.64.0.0/10
```

### 5. 可以复用同一个 auth key 和 state-dir 吗

不建议。

每台设备都应该有独立的：

```text
hostname
auth-key
state-dir
```

`state-dir` 是设备身份，不要从手机复制到电脑，也不要多个设备共用。

### 6. Windows 官方 Tailscale 客户端要不要开

二选一即可。

如果你用 Windows 官方 Tailscale 客户端，家里网段可以直接走系统 Tailscale 路由，mihomo 里就不一定需要 `type: tailscale`。

如果你想保持“所有规则都在 Clash Party/mihomo 里管理”，就关掉官方 Tailscale 客户端，用 mihomo 内置 `Tailscale-Home`。

Android 则更推荐 mihomo 内置 Tailscale，因为官方 App 会占用 VPNService。

## 原理总结

这套方案的关键不是“全局代理”，而是“按网段接管”。

家里 Debian 负责把传统内网变成 tailnet 可达的 subnet：

```text
192.168.1.0/24
192.168.0.0/24
```

手机和电脑上的 mihomo 负责把访问这些网段的流量送进内置 Tailscale：

```text
192.168.1.x -> Home LAN -> Tailscale-Home
192.168.0.x -> Home LAN -> Tailscale-Home
100.x.y.z   -> Home LAN -> Tailscale-Home
```

Windows 上的 TUN 只负责把“不懂代理的软件”的家里内网流量送进 mihomo：

```text
FinalShell -> 192.168.1.x -> Mihomo TUN -> Tailscale-Home
```

普通流量不进这条 TUN 路由：

```text
普通网页
DNS
日常代理分流
```

它们继续按原来的 Clash Party 逻辑工作。

一句话总结：

```text
家里 Debian 负责发布内网路由；
mihomo 内置 Tailscale 负责接入 tailnet；
Windows TUN 只接管家里网段；
Android 不开官方 Tailscale App，避免 VPNService 冲突。
```

最终效果就是：人在校园网，直接访问 `192.168.1.1`，就像人在家里一样。

## 参考链接

- [mihomo v1.19.25 Release](https://github.com/MetaCubeX/mihomo/releases/tag/v1.19.25)
- [mihomo Tailscale 出站文档](https://wiki.metacubex.one/config/proxies/tailscale/)
- [mihomo TUN 文档](https://wiki.metacubex.one/config/inbound/tun/)
- [Tailscale Linux 安装文档](https://tailscale.com/docs/install/linux)
- [Tailscale subnet router 文档](https://tailscale.com/docs/features/subnet-routers)
- [Tailscale auth keys 文档](https://tailscale.com/kb/1085/auth-keys)
