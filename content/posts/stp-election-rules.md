+++
date = '2026-06-23T22:19:22+08:00'
draft = false
title = 'STP 生成树原理与报文详解'
tags = ["数通", "STP"]
description = "根据规则选举 RB、RP、BP，封闭所有无任务端口，最终形成树状拓扑。"
+++

> 根据规则选举 RB、RP、BP，封闭所有无任务端口，最终形成树状拓扑。

---

## 生成树构建

### 选举规则

![STP 选举规则图解](/assets/images/stp_RB_RP_BP选举流程-压缩.jpg)

### 数据传输路径

当数据传输时，路径一定是 **RP → BP → RP → BP** 的样子。

## 端口状态

### 端口状态表

| 状态  | 英文         | 接收BPDU | 发送BPDU | 学习MAC地址表 | 转发数据 |
| --- | ---------- | ------ | ------ | -------- | ---- |
| 未启用 | Disabled   |        |        |          |      |
| 阻塞  | Discarding | ✓      |        |          |      |
| 侦听  | Listening  | ✓      | ✓      |          |      |
| 学习  | Learning   | ✓      | ✓      | ✓        |      |
| 转发  | Forwarding | ✓      | ✓      | ✓        | ✓    |

### 状态转换图

![端口状态转换图](/assets/images/端口转化流程.jpg)

## BPDU报文

### 两种BPDU报文

1. Configuration BPDU（Type 0x00）
   源于RB，层层下发。<u>通常</u>包含RB的BID，上游交换机到RB的RPC，上游交换机的BID。后两者在传输中会逐跳变化。每两秒一次。

2. TCN BPDU（Type 0x80）
   源于变化，包括断开和连接（Forwarding）。

### 报文详解

正常情况下每两秒发送一次00报文，无80报文。

发生变化时。

**0s**：上游交换机检测到端口变化或者收到下游报文，会向RB发送80报文。下游交换机<u>若检测到</u>端口变化即刻选举候选Forwarding（如果有的话），随即向上游发送80报文。上游收到80报文需要向根报告并且随即向下游回复TCA（在00报文中作为附带信息发出）报文，否则下游将持续发送20秒，2秒一次共十次。这里的下游可以是到根为止的路上任何一台交换机。80报文单向传输，因此<u>若断联是上游发起</u>并且物理连接不变，下游无法知道变化，等Max Age Timer(20s)到期才会发送80报文并开始选举。

**下游发现问题+30s**：下游自主选举新的stp topo，30秒后新端口正式成为Forwarding，再次向上游发送80报文，同错误80报文一样需要逐级回复。
**RB每次收到80报文+35s**：RB在收到80报文后的35秒（Max Age 20s + Forward Delay 15s）内会让00报文携带MAC地址表更新通知，使所有交换机大幅缩短MAC地址表过期时间，加速MAC表更新速度。过程中RB会多次收到80报文，因此实际时间绝不是35秒。

---

## 缩写速查

| 缩写      | 英文       | 翻译     |
| ------- | -------- | ------ |
| R       | Root     | 根      |
| B       | Bridge   | 网桥、交换机 |
| P       | Port     | 端口     |
| ID      | ID       | ID     |
| P(RPC)  | Path     | 路径     |
| C       | Cost     | 开销     |
| P(BPDU) | Protocol | 协议     |
| D(BPDU) | Data     | 数据     |
| U(BPDU) | Unit     | 单元     |

---

## 一些小记

1. 选举BP使用选举RP同样的方法，这里会出现一个问题。先前选RP时，如果选到PID这一步，将看port priority和port number。在同一交换机中port number一定不同，而BP选举的区域是网段，port number可能相同。这意味着似乎可能出现两个一样的PID，但实际上如果两个port不在一台交换机上，那在BID比较时就会被排除，因此这里没有漏洞。
2. bridge priority实际上并不能完全使用16bit，实际存储位置只有高位4bit，因此priority必须是4096的倍数
3. Listening到Learning再到Forwarding，每次变化需要等待15秒，不再需要其为Forwarding立刻变回Discarding。

<div style="font-size: 0.75em; color: #888; margin-top: 2em;">本文完全手写，AI仅辅助排版</div>
