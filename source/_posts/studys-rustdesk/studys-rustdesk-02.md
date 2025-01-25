---
title: RustDesk源码学习笔记 02-分析会和服务的通信报文1
date: 2025-01-25 12:18:00
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
KeyExchange 新接口
HealthCheck 新接口

### 本章节内容
本章节将首先介绍RustDeskServerDemo，然后贴出RustDesk官方仓库Wiki中对RustDesk工作原理的解释，之后在分析注册节点和公钥的四种报文类型。
本章节分为以下小节:
1. RustDeskServerDemo介绍
2. 《How does RustDesk work?》
3. 注册节点服务分析
4. 注册公钥服务分析

相关链接：
《How does RustDesk work?》 https://github.com/rustdesk/rustdesk/wiki/How-does-RustDesk-work%3F
《Issues: Working principle》 https://github.com/rustdesk/rustdesk/issues/594#issuecomment-1138342668
《RustDesk 源码阅读》 https://cloud.tencent.com/developer/article/1897847

## 正文
### RustDeskServerDemo介绍
RustDeskServerDemo是RustDesk早期开源的一个RustDesk服务端实现Demo
源码：https://github.com/rustdesk/rustdesk-server-demo

RustDeskServerDemo监听2个端口号，21116(UDP+TCP)和21117(TCP)。
21116用于节点注册，21117用于中继转发视频，音频流。
21116对应会和服务，21117对应中继服务。
与RustDeskServer不同，RustDeskServerDemo所有的流量都通过中继服务转发。

会和服务，处理以下几种请求
1. 节点注册（UDP），接受客户端发送的RegisterPeer请求，并做RegisterPeerResponse响应
2. 节点公钥注册（UDP），接受客户端发送的RegisterPk请求，并作RegisterPkResponse响应
3. 打洞请求（TCP），接受客户端A发送的PunchHoleRequest请求，向客户端B发送RequestRelay请求，告知客户端B中继服务的ip地址
4. 中继响应（TCP），接受客户端B返回的RelayResponse响应，将响应透传给客户端A，告知客户端A中继服务的ip地址

中继服务，在会和服务处理完中继响应之后，客户端A和B会分别和中继服务建立TCP连接，通过中继服务转发流量。

### 《How does RustDesk work?》
![](/img/posts/studys-rustdesk/02-01-working-principle.png "Working Principle")
![](/img/posts/studys-rustdesk/02-02-working-principle-2.png "Working Principle 2")

### 注册节点服务分析
#### 报文定义
``` protobuf
message RegisterPeer {
  string id = 1;
  int32 serial = 2;
}
message RegisterPeerResponse {
  bool request_pk = 2;
}
```
#### 介绍
RegisterPeer.id 客户端唯一标识符
RegisterPeer.serial 客户端版本号，用于标记客户端是否需要更新配置
RegisterPeerResponse.request_pk 客户端是否需要注册公钥

由客户端生成唯一标识符id，id将标记一个客户端节点Peer，Peer信息被存储在sqllite中，并用HashMap缓存。

##### Peer定义
``` rust
pub(crate) struct Peer {
  pub(crate) socket_addr: SocketAddr,
  pub(crate) last_reg_time: Instant,
  pub(crate) guid: Vec<u8>,
  pub(crate) uuid: Bytes,
  pub(crate) pk: Bytes,
  // pub(crate) user: Option<Vec<u8>>,
  pub(crate) info: PeerInfo,
  // pub(crate) disabled: bool,
  pub(crate) reg_pk: (u32, Instant), // how often register_pk
}
```

会和服务器处理RegisterPeer请求时:
第一次被注册节点或者id对应节点的ip地址发生变化时，需要重新注册公钥。
若服务器认为需要重新注册公钥时，通过RegisterPeerResponse通知客户端，要求客户端发起注册公钥的请求
若服务端发现客户端需要更新配置时，通过ConfigUpdate通知客户端配置变更

🔔注意： 当id注册时，注册信息不会立刻生效。


### 注册公钥服务分析
#### 报文定义
``` protobuf
message RegisterPk {
  string id = 1;
  bytes uuid = 2;
  bytes pk = 3;
  string old_id = 4;
}

message RegisterPkResponse {
  enum Result {
    OK = 0;
    UUID_MISMATCH = 2;
    ID_EXISTS = 3;
    TOO_FREQUENT = 4;
    INVALID_ID_FORMAT = 5;
    NOT_SUPPORT = 6;
    SERVER_ERROR = 7;
  }
  Result result = 1;
  int32 keep_alive = 2;
}
```
#### 介绍
RegisterPk.id 客户端唯一标识符
RegisterPk.uuid 客户端唯一标识符2
RegisterPk.pk 客户端公钥
RegisterPk.old_id 没有用到
RegisterPkResponse.result 注册公钥结果
RegisterPkResponse.keep_alive 存活时间，新字段，目前在RustDeskServer中没看到实现

会和服务器处理RegisterPk请求时:
如果没有送正确的uuid会导致注册失败
如果同时更换ip和pk，否则注册失败
更新成功后，服务器会记住Peer信息
