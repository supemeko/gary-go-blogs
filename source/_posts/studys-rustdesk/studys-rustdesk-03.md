---
title: RustDesk源码学习笔记 03-分析会和服务的通信报文2
date: 2025-01-27 23:25:00
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
本章节主要分析主控客户端如何通过PunchHoleRequest建立与被控客户端的连接，并分析相关报文
本章节分为以下小节:
1. 打洞请求流程分析
2. PunchHoleRequest 分析
3. FetchLocalAddr 分析
4. PunchHole 分析
5. LocalAddr 分析
6. RelayResponse 分析
7. PunchHoleSent 分析

## 正文
### 打洞请求流程分析
#### 会和服务器通知被控端处理
会和服务器接收到PunchHoleRequest时，会判断是否同一网段，是否使用NAT等细节。
如果主控和被控客户端，被判断为同一网段，则会发送FetchLocalAddr至被控端，否则发送PunchHole至被控端。

#### 被控服务器处理
当会和服务器向被控客户端发送FetchLocalAddr时，被控客户端会根据实际情况选择采用本地连接还是中继服务。
如果被控客户端选择本地服务，则向会和服务器发送LocalAddr，发送完成后，等待主控客户端建立连接。其中有个细节是LocalAddr通过新的TCP请求发送，发送完成后关闭TCP连接，并监听原端口等待主控客户端建立连接;
如果被控客户端选择中继服务，则向会和服务器发送RelayResponse，发送完成后建立与中继服务器的连接。

当会和服务器向被控客户端发送PunchHole时，被控客户端会根据实际情况选择采用中继服务还是其他模式。
如果被控客户端选择中继服务，则向会和服务器发送RelayResponse，发送完成后建立与中继服务器的连接。
如果被控客户端选择其他模式，则向会和服务器发送PunchHoleSent，发送完成后，等待主控客户端建立连接。

#### 会和服务器响应被控端处理
当会和服务器接收到由被控端发送的LocalAddr时，响应PunchHoleResponse至主控端
当会和服务器接收到由被控端发送的RelayResponse时，向主控端透传RelayResponse
当会和服务器接收到由被控端发送的PunchHoleSent时，响应PunchHoleResponse至主控端

#### 主控端收到会和服务器PunchHoleRequest处理结果
当主控服务器收到会和服务器PunchHoleResponse和RelayResponse时，对应向被控端建立TCP连接

### PunchHoleRequest 分析
#### 报文定义
``` protobuf
message PunchHoleRequest {
  string id = 1;
  NatType nat_type = 2;
  string licence_key = 3;
  ConnType conn_type = 4;
  string token = 5;
  string version = 6;
}
```
#### 介绍
PunchHoleRequest.id 被控客户端唯一标识符
PunchHoleRequest.nat_type 中继类型
PunchHoleRequest.licence_key 会和服务器公钥
PunchHoleRequest.conn_type RustDeskServer中未使用，不分析
PunchHoleRequest.token RustDeskServer中未使用，不分析
PunchHoleRequest.version RustDeskServer中未使用，不分析

用于主控端向会和服务器请求被控客户端的连接方式。主控端向会和服务器发送该请求后，可能会收到来自会和服务器的两种应答报文。分别是RelayResponse和PunchHoleResponse，根据这两种报文提供的信息以及指示，主控端将发起与被控端通过中继服务器的连接，或者与被控端的直接连接。

### FetchLocalAddr 分析
#### 报文定义
``` protobuf
message FetchLocalAddr {
  bytes socket_addr = 1;
  string relay_server = 2;
}
```
#### 介绍
FetchLocalAddr.socket_addr 主控端在会和服务器网络环境中的ip地址
FetchLocalAddr.relay_server 会和服务器推荐使用的中继服务的ip地址

用于在会和服务器发现主控端可能可以与被控端处于同一内网连接时，会和服务器向被控端请求建立主控端与被控端连接。

### PunchHole
#### 报文定义
```
message PunchHole {
  bytes socket_addr = 1;
  string relay_server = 2;
  NatType nat_type = 3;
}
```
#### 介绍
PunchHole.socket_addr 主控端在会和服务器网络环境中的ip地址
FetchLocalAddr.relay_server 会和服务器推荐使用的中继服务的ip地址
FetchLocalAddr.nat_type 主控端或会和服务器要求使用的NAT方式

用于会和服务器向被控端请求建立主控端与被控端连接。

### LocalAddr 分析
#### 报文定义
```
message LocalAddr {
  bytes socket_addr = 1;
  bytes local_addr = 2;
  string relay_server = 3;
  string id = 4;
  string version = 5;
}
```
#### 介绍
LocalAddr.socket_addr 主控端在会和服务器网络环境中的ip地址
LocalAddr.local_addr 被控端的本地ip地址
LocalAddr.relay_server 被控端选择的中继服务器地址
LocalAddr.id 被控端的唯一标识符
LocalAddr.version 被控客户端版本号

用于被控端与主控端处于同一内网时，被控端通知会和服务器和主控端，可使用内网直连方式连接。

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
涉及其他流程，故下节分析

### PunchHoleSent 分析
#### 报文定义
``` protobuf
message PunchHoleSent {
  bytes socket_addr = 1;
  string id = 2;
  string relay_server = 3;
  NatType nat_type = 4;
  string version = 5;
}
```

#### 介绍
PunchHoleSent.socket_addr 主控端在会和服务器网络环境中的ip地址
PunchHoleSent.id 被控端的唯一标识符
PunchHoleSent.relay_server 被控端推荐使用的中继服务器地址
PunchHoleSent.nat_type 被控端推荐使用的NAT方式
PunchHoleSent.version 被控客户端版本号

用于被控端决定不能使用中继服务时，通知会和服务器和主控端采用某种连接方式。
