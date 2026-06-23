+++
date = '2026-06-23T22:19:22+08:00'
draft = false
title = 'STP 生成树：RB、RP、BP 选举规则速记'
tags = ["数通", "STP"]
description = "根据规则选举 RB、RP、BP，封闭所有无任务端口，最终形成树状拓扑。"
+++

> 根据规则选举 RB、RP、BP，封闭所有无任务端口，最终形成树状拓扑。

---

## 选举规则

![STP 选举规则图解](/assets/images/stp-election-rules.jpg)

## 数据传输路径

当数据传输时，路径一定是 **RP → BP → RP → BP** 的样子。

---

## 缩写速查

| 缩写 | 英文    | 翻译       |
|------|---------|------------|
| R    | Root    | 根         |
| B    | Bridge  | 网桥、交换机 |
| P    | Port    | 端口       |
| ID   | ID      | ID         |
| P    | Path    | 路径       |
| C    | Cost    | 开销       |

---

## 一个小细节

选举 BP 使用选举 RP 同样的方法，这里会出现一个问题。

先前选 RP 时，如果选到 **PID** 这一步，将看 port priority 和 port number。在同一交换机中 port number 一定不同，因此不会出现平局。

而 **BP 选举的区域是网段**，port number 可能相同——乍一看似乎可能出现两个一样的 PID。但实际上，如果两个 port 不在同一台交换机上，在 **BID 比较**那一步就已经分出胜负了，根本走不到 PID 这一步。因此这里没有漏洞。
