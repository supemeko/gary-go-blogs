---
title: RustDesk源码学习笔记 07-关键类型，解决一个issue和结语
date: 2025-05-03 02:47:01
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
1. 从关键类型观察rustdesk中各模块的关系，各个模块是如何通信和整合到一起的。
2. rust flutter绑定
3. 解决一个issue

### 结语
这个系列的最后一篇笔记了，剩下都是些业务代码，参考意义不大。
为了证明学习没有烂尾，这边解决一个ISSUE证明学习过。

## 正文
### 从关键类型观察rustdesk中各模块的关系，各个模块是如何通信和整合到一起的。
struct Server; // 作为被控端管理服务，连接
trait Service; // 被控端的服务，如剪切板，声音等
struct Config; // 管理部分配置信息
struct Connection; // 作为被控端，具体管理连接
struct ConnInner;  // 作为被控端，作为连接的副本提供给Service接受视频和信息流
struct ServiceInner; // 作为被控端，包含Service使用到的公共字段
struct RendezvousMediator; //用于处理会合服务相关信息交互
trait Interface; //客户端能力的抽象
trait InvokeUiSession; //用于对象消息传递,主要从rust业务代码到ui对象,如接受服务器控制指令,向flutter界面传递. 有时也用于flutter界面向rust端请求信息

### rust flutter绑定
#### 代码生成
项目使用flutter_rust_bridge 1.80.1版本 用于生成flutter rust绑定的相关代码，涉及以下文件：
- src/flutter_ffi.rs  //映射文件
- src/bridge_generated.io.rs //自动生成的代码
- src/bridge_generated.rs //自动生成的代码
- flutter/lib/generated_bridge.dart //自动生成的代码
- flutter/lib/generated_bridge.freezed.dart //自动生成的代码

#### 概述
dart方面主要涉及：
- struct RustdeskImpl; //dart对象，是rust的绑定，包含所有功能
rust方面主要是通过flutter.rs 操作 flutter_ffi.rs
flutter.rs中FlutterHandler实现了InvokeUiSession，使得主控端的控制指令可以到达flutter

#### 流
flutter_rust_bridge提供同步调用的方式,以及提供流来传递信息。
在dart表现为 stream.listen((message){})
在rust表现为 StreamLink<T>.add(T message) 

#### 事件
rust里面实现了注册事件，取消注册事件，根据事件名称来路由等功能，经典的时间处理模式。


### 解决一个issue
#### 概述
issue地址： https://github.com/rustdesk/rustdesk/issues/10005
1. rustdesk中支持通过多种地址名称对应不同的方式建立连接
   - id （与其他桌面连接软件相同）
   - ip地址直连 （好评如潮，我主要使用这种方式，ipv6直连真爽）
   - id + 私服地址
2. rustdesk提供收藏节点的功能，客户端会定期请求收藏节点的在线状态

问题出在虽然rustdesk允许通过“id + rustdesk服务器地址”控制电脑，但是这种方式连接的节点但不支持查询在线状态

#### 解决方案
观察到rustdesk每次都建立新的请求获取节点在线状态且使用统一入口查询节点信息。
```rust
async fn query_online_states_(
  ids: &Vec<String>,
  timeout: std::time::Duration,
) -> ResultType<(Vec<String>, Vec<String>)> {
  //...
}
```
函数签名如上，这就是需要修改的位置。

```rust 
//...
let mut socket = match create_online_stream().await {
  Ok(s) => s,
  Err(e) => {
      log::debug!("Failed to create peers online stream, {e}");
      return Ok((vec![], ids.clone()));
  }
};
// 使用socket发送请求
```
query_online_states_ 函数内包含以上代码

```rust
//...
let (rendezvous_server, _servers, _contained) = crate::get_rendezvous_server(READ_TIMEOUT).await;
// 使用 rendezvous_server
```
而create_online_stream 函数则从全局变量取出服务器地址。

```rust
async fn query_online_states_(
  ids: &Vec<String>,
  timeout: std::time::Duration,
  rendezvous_server: String,
) -> ResultType<(Vec<String>, Vec<String>)> {
  let mut socket = match create_online_stream(rendezvous_server).await {
    Ok(s) => s,
    Err(e) => {
        log::debug!("Failed to create peers online stream, {e}");
        return Ok((vec![], ids.clone()));
    }
  };
}
```
所以问题就很简单了，将query_online_states_函数定义为：
  输入 "一组ids""和"rendezvous_server"组合
  输出 在线的id集合,不在线的id集合

```rust
let ids: Vec<String> = ...;
let groups: HashMap<String, Vec<String>> = ids.iter().map(...).fold(HashMap:new(), || {...});
for group in groups {
  let (on, off) = query_online_states_(group, Timeout)
  // 使用on和off
}
```
1. 对ids分组获得多组"一组ids""和"rendezvous_server"的组合，
2. 应用函数query_online_states_，分别得到每组的结果。
3. 将结果合并后返回

#### 技术细节
ids是 "id"和"id@服务器地址" 的组合，首先需要将id和服务器地址分离出来。
这里需要找项目中已有的方法，避免代码重复以及实现不统一的问题，这个工作挺费劲的。
然后向服务器查询的是分离出来的id,但是结果要用id@服务器地址回报。
所以需要将groups改为{ server: { pure_id, id }[] }[]，其中id = pure_id@server
结果合并的时候，需要将server返回的 pure_id 转换回 id

#### 最后欣赏一下自己写的代码
https://github.com/supemeko/rustdesk/commit/4432a3fef9c5e0f5dbc15626c0ac38500b8b4e66
写的挺好的，就是clone有点多，然后有些细节没有补充，比方说纯ip访问其实可以选择直接过滤掉。

