---
title: RustDesk源码学习笔记 05-分析会和服务的通信报文4
date: 2025-02-02 17:32:00
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
本章节粗略的过一遍剩余的报文类型
本章节分为以下小节:
1. ConfigUpdate 分析
2. SoftwareUpdate 分析
3. TestNatRequest和TestNatResponse 分析
4. PeerDiscovery 分析
5. OnlineRequest和OnlineResponse 分析

## 正文
### ConfigUpdate 分析
#### 报文定义
```proto
message ConfigUpdate {
  int32 serial = 1;
  repeated string rendezvous_servers = 2;
}
```
#### 介绍
1. 若会和服务器发现客户端需要更新配置时，通过ConfigUpdate通知客户端配置变更
2. TestNatResponse报文中也携带了ConfigUpdate，作用相同

### SoftwareUpdate 分析
#### 报文定义
```proto
message SoftwareUpdate {
  string url = 1;
}
```
#### 介绍
向会和服务器发送SoftwareUpdate时，会和服务器通过SoftwareUpdate返回软件url

### TestNatRequest和TestNatResponse 分析
#### 报文定义
```proto
message TestNatRequest {
  int32 serial = 1;
}
message TestNatResponse {
  int32 port = 1;
  ConfigUpdate cu = 2; // for mobile
}
```
#### 介绍
客户端向会和服务器发送TestNatRequest时，会和服务器通过TestNatResponse返回中继服务列表和客户端请求到服务端的端口号

### PeerDiscovery 分析
#### 报文定义
```proto
message PeerDiscovery {
  string cmd = 1;
  string mac = 2;
  string id = 3;
  string username = 4;
  string hostname = 5;
  string platform = 6;
  string misc = 7;
}
```
#### 介绍
用于查询同一网络下的其他客户端

### OnlineRequest和OnlineResponse 分析
#### 报文定义
```proto
message OnlineRequest {
  string id = 1;
  repeated string peers = 2;
}
message OnlineResponse {
  bytes states = 1;
}
```
#### 介绍
用于查询若干个节点的状态
