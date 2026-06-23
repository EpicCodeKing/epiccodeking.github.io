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

## 一些小记

1. 选举BP使用选举RP同样的方法，这里会出现一个问题。先前选RP时，如果选到PID这一步，将看port priority和port number。在同一交换机中port number一定不同，而BP选举的区域是网段，port number可能相同。这意味着似乎可能出现两个一样的PID，但实际上如果两个port不在一台交换机上，那在BID比较时就会被排除，因此这里没有漏洞。
2. 补充：Bridge Priority 实际上并不能完全使用 16 bit，实际存储位置只有高位 4 bit，因此 priority 必须是 4096 的倍数。

<div style="font-size: 0.75em; color: #888; margin-top: 2em;">本文完全手写，AI仅辅助排版</div>
