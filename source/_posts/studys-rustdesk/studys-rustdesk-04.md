---
title: RustDesk源码学习笔记 04-分析会和服务的通信报文3
date: 2025-02-02 16:53:00
updated:
tags:
  - "文章"
  - "学习笔记"
categories:
  - "学习笔记"
  - "RustDesk源码学习笔记"
description: "学习一下RustDesk开源项目，试图了解udp，tokio，p2p,rdp的一些知识"
cover: https://jsd.012700.xyz/gh/jerryc127/CDN@latest/cover/default_bg.png
---

## 介绍
### 概要
RustDesk分为服务端和客户端，服务端有2种服务器，会和服务器hhbs，中继服务器hbbr
在会和服务器，中继服务器，客户端之间使用按protobuf定义的报文格式进行节点管理
在这些负责节点管理的报文中，初步判断共有19种报文类型，接下来的几个章节会对这19种报文的作用逐一分析。
包括（按字面意思翻译）：
RegisterPeer 注册节点
RegisterPeerResponse 注册节点响应
PunchHoleRequest 打洞请求
PunchHole 打洞
PunchHoleSent 打洞送出
PunchHoleResponse 打洞响应
FetchLocalAddr 拉取本地地址
LocalAddr 本地地址
ConfigUpdate 配置更新
RegisterPk 注册公钥
RegisterPkResponse 注册公钥响应
SoftwareUpdate 软件更新
RequestRelay 请求中继
RelayResponse 中继响应
TestNatRequest 测试NAT请求
TestNatResponse 测试NAT响应
PeerDiscovery 节点发现
OnlineRequest 在线请求
OnlineResponse 在线响应
KeyExchange 新接口,未计入19种报文类型
HealthCheck 新接口,未计入19种报文类型

### 本章节内容
本章节主要分析RequestRelay和RequestRelay的相关报文
本章节分为以下小节:
1. RequestRelay的报文格式
2. RequestRelay流程分析
3. RelayResponse 分析

## 正文
### RequestRelay的报文格式
```protobuf
message RequestRelay {
  string id = 1;
  string uuid = 2;
  bytes socket_addr = 3;
  string relay_server = 4;
  bool secure = 5;
  string licence_key = 6;
  ConnType conn_type = 7;
  string token = 8;
}
```
请求会和服务时:
RequestRelay.id 被控端id
RequestRelay.uuid 请求id
RequestRelay.token 访问token
RequestRelay.secure 是否安全连接
RequestRelay.relay_server 中继服务器的ip地址
RequestRelay.socket_addr 主控端在会和服务器网络环境的ip地址

请求中继服务时:
使用相同的uuid的两个客户端，将建立tcp连接

### RequestRelay流程分析
主控发送RequestRelay至会和服务，会和服务将其转发至对应被控端，等待RelayResponse，完成RelayResponse后请求建立中继连接
被控端返回RelayResponse至会和服务，会和服务将其转发至主控端，并请求建立中继连接

### RelayResponse 分析
#### 报文定义
```
message RelayResponse {
  bytes socket_addr = 1;
  string uuid = 2;
  string relay_server = 3;
  oneof union {
    string id = 4;
    bytes pk = 5;
  }
  string refuse_reason = 6;
  string version = 7;
  int32 feedback = 9;
}
```
#### 介绍
RelayResponse.socket_addr 主控端地址
RelayResponse.uuid 请求id

在PunchHoleRequest流程和RequestRelay流程中
用于提示主控端，被控端已经准备向中继服务建立连接了。
