---
title: "Tinykv（二）Raft KV"
date: 2022-05-04T20:43:11+08:00
categories: [tinykv]
tags: [go, raft, database]
draft: true
---

Project2 需要实现一个基于 [raft](https://raft.github.io/raft.pdf) 的 kv 服务，主要分为三部分：

-   实现 raft 算法
-   在 raft 之上构建一个容错的 kv 服务
-   添加 raft 日志的垃圾回收和快照支持

## Raft

raft 算法的实现也可以分为三部分：

-   Leader 选举
-   日志复制
-   安全性

## 算法结构

tinykv 使用状态机来实现 raft 算法，主要由 tick 和 Step 函数驱动 raft 节点的工作，单个节点是顺序执行的，不存在并行操作，大大降低了复杂度。

tick：由上层应用调用，实际上充当时钟的作用，驱动节点的选举和心跳。

Step：状态机，由上层调用，输入为 message 和节点的状态（Follower、Candidate、Leader），根据输入执行相应的操作。

### MsgType

tinykv 将 message 分为两类：本地消息和非本地消息。本地消息是自己发给自己的，特点是任期为 0。

#### MsgHup

本地消息，收到后发起选举。

| 所需字段 | 备注     |
| -------- | -------- |
| MsgType  | 消息类别 |

#### MsgBeat

本地消息，只有 Leader 响应，收到后广播心跳。

| 所需字段 | 备注     |
| -------- | -------- |
| MsgType  | 消息类别 |

#### MsgPropose



#### MsgAppend



#### MsgAppendResponse



#### MsgRequestVote

候选者收到后发送投票请求。

| 所需字段 | 备注                |
| -------- | ------------------- |
| MsgType  | 消息类别            |
| Term     | 候选者 Term         |
| Index    | 论文中 lastLogIndex |
| LogTerm  | 论文中 lastLogTerm  |
| From     | 候选者              |
| To       | 目标节点            |

#### MsgRequestVoteResponse

回复 MsgRequestVote，需要满足选举限制。

| 所需字段 | 备注          |
| -------- | ------------- |
| MsgType  | 消息类别      |
| Term     | 节点当前 Term |
| Reject   | 是否投票      |
| To       | 目标节点      |

#### MsgSnapshot



#### MsgHeartbeat

Leader 发送心跳。

| 所需字段 | 备注                                 |
| -------- | ------------------------------------ |
| MsgType  | 消息类别                             |
| Term     | Leader 的 Term                       |
| Commit   | min(matchIndex, r.RaftLog.committed) |
| To       | 目标节点                             |

Follower 收到后根据 Commit 更新自己的 commited.

#### MsgHeartbeatResponse



#### MsgTransferLeader



#### MsgTimeoutNow
