+++
date = '2026-07-26T20:51:29+08:00'
draft = false
title = 'OSPF 协议简谈'
tags = ["数通", "OSPF"]
description = "以邻居状态为轴理解 OSPF 邻接建立过程，涵盖报文类型、LSA 分类与区域间路由。"
+++

> 以邻居状态为轴理解 OSPF 邻接建立过程，涵盖报文类型、LSA 分类与区域间路由。

---

## 简谈

OSPF 的笔记不再整理，仅在此简述大致过程，读者再看笔记理解细节：

> 区域内部：理解应以**邻居状态**为轴，两台路由器从 Down 到 Full 完成两台机器的同步。本文的连接过程省略了 Loading 状态，此状态下路由器用第一类 LSA 数据搭载于 LSR 报文上进行同步。线路生产者定时泛洪自己产生的线路。定时清除没有更新的线路。有新变化立刻将变化内容泛洪到全局。
>
> 区域之间：区域之间不传递具体链路信息，具体到表现就是，网络内部路由器只知道某个网段要往某台边界路由器走，不再关心边界路由器之后的路线。

---

## 杂记

router-id：全局唯一、本质是 32 位二进制，实际用点分十进制表示（IPv4 的 IP 格式）、建议手动配置

cost（开销）：100Mbps / 接口带宽，向上取整。100Mbps 是参考值，可修改

监听地址：224.0.0.5

### 传输报文类型

![传输报文类型](/assets/images/ospf-msg-types.png)

### 邻居状态

![邻居状态](/assets/images/ospf-neighbor-states.png)

对于组播网络，非 DR/BDR 之前会停留在 2-Way 阶段

### OSPF 公共报文头

![OSPF 公共报文头](/assets/images/ospf-header.png)

### Hello

![Hello 报文](/assets/images/ospf-hello.png)

Hello：除帧中继和虚链路外，都是 224.0.0.5 组播发送。其中包含邻居列表。不触发发送，周期性发送。

### DBD

![DBD 报文](/assets/images/ospf-dbd.png)

flags-init 只在第一次发送时为 1，用于开启主从判断

flags-ms 在未知前都为 1，知道主从后从方改为 0

**LSA 摘要条目**

![LSA 摘要条目](/assets/images/ospf-lsa-header.png)

### 常见 LSA 类型

![常见 LSA 类型](/assets/images/ospf-lsa-types.png)

### 周期性

Hello 报文：周期且仅周期发送。

LSU 报文：周期或触发发送。每 30 分钟触发一次，将自身产生的 LSA 泛洪出去；当自身 LSA 变化时触发。

LSA 老化：每 60 分钟清除老化 LSA。

### 从 Down 到 Exchange

初始状态：R1 (RID=1.1.1.1) 和 R2 (RID=2.2.2.2) 直连，OSPF 刚启用，互不知道对方存在

┌─ 阶段：Down → Init ────────────────────────────────────────────────────┐
│
│ t=0s     R2 周期性发 Hello（Neighbors=空）
│          └─ R1 收到 → R1 变 Init
│
│ t=1s     R1 周期性发 Hello（Neighbors=空）
│          └─ R2 收到 → R2 变 Init
│
├─ 阶段：Init → 2-Way ───────────────────────────────────────────────────┤
│
│ t=10s    R2 下一次周期 Hello（Neighbors=[1.1.1.1]）
│          └─ R1 收到 → R1 变 2-Way
│
│ t=11s    R1 下一次周期 Hello（Neighbors=[2.2.2.2]）
│          └─ R2 收到 → R2 变 2-Way
│
├─ 阶段：2-Way → ExStart ────────────────────────────────────────────────┤
│
│ "对方知道自己"且"自己告知对方自己知道对方"后，各自进入 ExStart，同时发
│ 第一个 DBD。这个过程紧接着 2-Way。甚至只要处于 Init（收到过对方一次不
│ 含自己的 Hello），收到 DBD 也会直接从 Init 进 ExStart，跳过 2-Way。
│
│ R1 ──→ DBD (I=1, M=1, MS=1, Seq=X, MTU=1500, LSA=空)
│        "我是 Master，Router ID 1.1.1.1，序列号 X"
│
│ R2 ──→ DBD (I=1, M=1, MS=1, Seq=Y, MTU=1500, LSA=空)
│        "我是 Master，Router ID 2.2.2.2，序列号 Y"
│
│ 双方从 DBD 包头取出 Router ID 比较：
│        2.2.2.2 > 1.1.1.1  →  R2 = Master，R1 = Slave
│        序列号 Y 成为正式序列号
│
├─ 阶段：ExStart → Exchange ─────────────────────────────────────────────┤
│
│ R1（Slave）认怂，等 Master 先动。
│
│ R2 ──→ DBD (I=0, M=1, MS=1, Seq=Y+1, +LSA 摘要)
│        "这是我的 LSDB 目录，还有更多"
│        ※ RFC 规定 Master 首次进 Exchange 应先递增（Seq=Y+1），部分
│           实现第一轮不递增（Seq=Y）。Slave 无脑跟，不影响连通。
│
│ R1 ──→ DBD (I=0, M=1, MS=0, Seq=Y+1, +LSA 摘要)
│        "这是我这边的目录，用你的序列号"
│
│ R2 ──→ DBD (I=0, M=1, MS=1, Seq=Y+2, +LSA 摘要)
│        "下一批，还有"
│
│ R1 ──→ DBD (I=0, M=1, MS=0, Seq=Y+2, +LSA 摘要)
│        "收到，我这边也还有"
│
│ ... Master 轮询驱动，每次 Seq+1 ...
│
│ R2 ──→ DBD (I=0, M=0, MS=1, Seq=Y+n, +最后一批 LSA)
│        "我没更多了"
│
│ R1 ──→ DBD (I=0, M=0, MS=0, Seq=Y+n, +最后一批 LSA)
│        "我也没了"
│
│ 规则：先发完的一方设 M=0。Master 发完后若 Slave 仍 M=1，则持续发空
│ DBD（M=0）轮询；Slave 发完后若 Master 仍 M=1，则每次回空 DBD（M=0）
│ 应答。双方 M 都为 0 后 Exchange 结束。
│
└─ Exchange 结束 ─────────────────────────────────────────────────────────┘

结果：
  R1 知道 R2 有哪些 LSA，R2 知道 R1 有哪些 LSA
  → 发现对方有自己缺的 LSA → 进 Loading，发 LSR 索取
  → 目录完全一致 → 直接跳 Full

---

本次笔记结合人工和 AI，其中"从 Down 到 Exchange"由 AI 生成，人工逐行核验。最终排版工作由 AI 完成。
