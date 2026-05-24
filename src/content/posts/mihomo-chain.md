---
title: 利用 SubStore 生成 Mihomo 全平台通用链式代理配置
published: 2026-01-01
description: 借助 SubStore 的多订阅聚合与脚本注入能力，构建基于 Mihomo 的现代化链式代理（中转 → 落地）方案，实现全平台一致体验
image: "/images/posts/mihomo-chain/mihomo-chain-cover.png"
tags: [VPS,工具,教程]
category: 教程
draft: false
---

## 💡 前言：为什么我们需要“链式代理”？


对于追求**稳定性、低延迟与低风控**的用户来说，代理方案往往面临一个经典两难：

- **自建落地 VPS**
  - ✅ IP 独享、干净，Netflix / ChatGPT / Cloudflare 友好
  - ❌ 多为国际线路优化，直连回国绕路严重，延迟与丢包明显，无法直接使用

- **机场节点**
  - ✅ IEPL / IPLC / 公有云入口，延迟低、路径优秀
  - ❌ 多人共享出口 IP，风控频繁，验证不断
- **优质直连 VPS**
  - ✅ 线路优秀，甚至有超越专线的延迟水平
  - ❌ 价格高昂，富哥专属

**链式代理（Chain Proxy）** 的目标正是结合三者的优缺点：

> **使用价格相对低廉的机场节点作为中转入口（Relay），  
> 再将流量转发至自建落地节点（Exit）访问目标网站**

最终的流量路径为：

> 本地 → 中转节点（机场） → 独享落地节点（VPS） → 目标网站

过去， Mihomo 常通过 `relay` 协议组实现链式代理，但随着内核演进，该方案已被弃用  
本文将基于 **最新的 `dialer-proxy / underlying-proxy` 机制**，结合 **SubStore 的脚本注入能力**，构建一套**现代、稳定、可维护**的链式代理方案  
::github{repo="sub-store-org/Sub-Store"}  

---

## 🧬 核心逻辑

本方案的核心理念是：  
**将复杂的链式逻辑完全后移至 SubStore 后端管理，客户端只负责选择策略**

具体做法是，在 SubStore 中创建 **两个完全独立、职责清晰的订阅池**

- **中转订阅（Relay）**
  - 机场节点 / 专线入口
  - 只负责“进来快”

- **落地订阅（Exit）**
  - 自建 VPS 节点
  - 只负责“出去稳”

通过脚本注入，在 **落地节点上声明其出站必须先经过 Relay 策略组**，从而实现链式转发

:::danger[重要注意事项]
- **两个订阅池中严禁出现名称重复的节点**
- **Relay 与 Exit 不可互相包含**

否则将导致 Mihomo 内核出现死循环（Loop），表现为内存占用Boom！
:::

---

## 🛠️ 第一步：准备工作

在开始之前，请确保你已具备以下条件：

- 已部署并可正常使用 **SubStore**
- 至少一条 **机场/自建订阅**（用于中转）
- 至少一条 **自建落地节点**（如 `ss://`、`vless://` 等）
- 基本的 SubStore 操作认知
- 以及一颗乐于折腾的心 🧠

---

## 🛠️ 第二步：在 SubStore 中创建订阅
:::tip[提示]
如果你有多个机场，可先分别添加为单条订阅，再通过 **组合订阅** 聚合为一个节点池，便于统一管理
:::

### 1️⃣ 创建「纯中转」订阅（Relay）

在 SubStore 首页点击左上角 **➕**，选择「单条 / 组合订阅」：

- **名称**：`Mihomo-chain-relay`
- **手动选择的订阅**：勾选你的机场订阅
- **不需要**注入脚本
- 可按需使用正则过滤流量节点、剩余信息等

**目标**：  
确保该订阅中 **只包含可作为跳板的中转节点**

---

### 2️⃣ 创建「纯落地」订阅（Exit）

再次点击 **➕**，创建第二条订阅：

- **名称**：`Mihomo-chain-exit`
- **手动选择的订阅**：勾选你的自建落地节点
- **注入脚本类型**：`脚本`
- **脚本内容**（覆盖默认代码）：
```javascript
$server['underlying-proxy'] = "Relay"
```
`underlying-proxy` 指向的 `Relay` 必须是 Mihomo 配置文件中真实存在的策略组名称  
如果你在模板中修改了分组名，这里也必须同步修改，否则 Exit 节点将无法出站  
:::tip[提示]
脚本的作用是给落地节点添加"HK Relay"出站，告诉 Mihomo：“任何使用本节点的流量，请先通过 `Relay` 策略组中转 ”
:::

---

### 3️⃣ 为订阅生成安全分享链接
为避免暴露后端订阅地址，建议通过**分享功能**生成独立链接
* 点击订阅右侧「⋯」→「分享」
* 设置有效期
* 使用高强度 Token（推荐于[1password](https://1password.com/zh-cn/password-generator)生成大于 20 位的高强度 Token）
![订阅创建分享链接](/images/posts/mihomo-chain/mihomo-chain-create-share.png)
![设置有效期](/images/posts/mihomo-chain/mihomo-chain-Set%20validity%20period.png)    


## 🛠️ 第三步：添加 Mihomo 配置模板
### 1️⃣ 添加配置文件
:::tip[提示]
可以使用我提供的示例模板  
* [https://raw.githubusercontent.com/ZekTy/mihomo-chain/refs/heads/main/yaml/mihomo-chain.yaml](https://raw.githubusercontent.com/ZekTy/mihomo-chain/refs/heads/main/yaml/mihomo-chain.yaml)  
或使用你自己的模板，但需确保  
* Relay / Exit 分组名称一致
* Provider 名称正确
:::

点击底部栏`第二个`图标(文件管理)，单击左上角加号，选择类型为`文件` -> `本地`，名称随意  
* `查询流量信息订阅链接`可填写机场订阅链接来获得流量信息  
* 将[配置模板](https://raw.githubusercontent.com/ZekTy/mihomo-chain/refs/heads/main/yaml/mihomo-chain.yaml)中的内容全部复制并粘贴至编辑框内  
![添加配置文件](/images/posts/mihomo-chain/mihomo-chain-flie.png)   

```yaml
proxy-providers:   #截取完整文件片头演示
  Mihomo-chain-relay:
    # ↓↓↓ 填入你生成的【纯中转】分享链接，保留双引号 ↓↓↓
    url: "填入分享管理内的中转节点链接"
    type: http
    interval: 86400
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
    proxy: DIRECT
  Mihomo-chain-exit:
    # ↓↓↓ 填入你生成的【纯落地】分享链接，保留双引号 ↓↓↓
    url: "填入分享管理内的落地节点链接"
    type: http
    interval: 86400
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 300
    proxy: DIRECT
```
保存配置后，单击刚才创建的文件，选择「通用订阅」获取最终链接，即可在各平台软件中导入使用  
## 🛠️ 第四步：在 Mihomo 中使用  
### 📱 全平台客户端食用指南

1. 粘贴上一步生成的通用订阅链接导入客户端
2. 进入代理界面：
* 在 Relay 分组：选择你要用于中转的机场节点
* 在 Exit 分组：选择你要用于最终访问网站的落地节点
3. 最后，在主 Proxy 分组选择 Exit，即刻开启丝滑的链式代理体验

### ✅ 兼容性
目前测试`Windows Clash Party` `Android Flclash` `Android box模块` `Openclash` `Nikki` `IOS Clash Mi`均可正常使用  

:::tip[Q&A]
如何确认链式代理已经生效？  
在zashboard面板中可以看见类型为`Inner`的连接，节点为你在`Relay`分组中所选；可看到各网页访问路径即为正常
![Inner](/images/posts/mihomo-chain/mihomo-chain-Inner.png)  
:::