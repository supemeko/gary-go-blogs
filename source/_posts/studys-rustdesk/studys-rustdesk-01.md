---
title: RustDesk源码学习笔记 01-实现一个基于tokio的UdpFramed的简单demo
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

## 写在前面
我会在这个系列学习笔记中，记录学习RustDesk开源项目的一些过程。
学习笔记当然就不只是包含RustDesk的源码，还包括学习源码过程中学习的其他知识。
那么现在开始～

## 介绍
### 概要
RustDesk中使用了UdpFramed，稍微学习了解一下，然后记录成这个系列第一篇笔记《01-实现一个基于tokio的UdpFramed的简单demo》

文章包含以下几个小节:
1. 实现一个多bin输出的rust项目；
Demo项目包含两个bin输出，一个客户端client.rs和一个服务端server.rs，所以先尝试实现一个多bin输出的rust项目；
2. 引入依赖，然后吐槽一下rust的futures
3. 谈一下rust的错误处理
4. 实现demo

## 正文
### 实现一个多bin输出的rust项目
#### 创建 Cargo.toml 文件:
``` toml Cargo.toml
[package]
name = "tokio_udp"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "client"
path = "src/bin/client.rs"

[[bin]]
name = "server"
path = "src/bin/server.rs"
```

#### 创建bin文件:
``` rust
// src/bin/server.rs
fn main() {
  println!("Hello from the server!");
}

// src/bin/client.rs
fn main() {
  println!("Hello from the client!");
}
```

#### 现在可以执行了
``` bash
# 运行 server
cargo run --bin server #Hello from the server!
# 运行 client
cargo run --bin client #Hello from the client!
```

#### 在server.rs和client.rs中使用共享代码
``` rust
// src/lib.rs
pub fn string() -> String {
  "Hello from the library!".to_string()
}

// src/bin/server.rs
use tokio_udp; // tokio_udp是Demo项目名称
fn main() {
  let msg = tokio_udp::string();
  println!("server: {}", msg); //server: Hello from the library!
}

// src/bin/client.rs
use tokio_udp; // tokio_udp是项目名称
fn main() {
  let msg = tokio_udp::string();
  println!("client: {}", msg); //client: Hello from the library!
}
```

#### 短评
AMAZING!现在已经实现了多bin输出的rust项目了☺ Gemini真厉害啊!

### 引入依赖，然后吐槽一下rust的futures
``` bash
cargo add tokio -F full
cargo add tokio-util  -F full
cargo add futures
```
引入依赖相比其他语言只是记录一个包名，rust还需要指定具体futures。
而且代码中无法自动引入未指定futures提供的symbol，导致有时候搞不清楚代码那里编译不过。
不过吃一堑长一智，其实用起来应该也不会太麻烦，但还是吐槽一下

### 谈一下rust的错误处理
``` src/lib.rs
pub type Error = Box<dyn std::error::Error>;
pub type Result<T> = core::result::Result<T, Error>;
```

``` rust
use tokio_udp; //tokio_udp是Demo项目名称
use tokio_udp::Result;
fn main() -> Result<()> {
  // 尽情使用 result!

  Ok(())
}
```
不想处理的错误直接向上传递即可，与try catch在易用性上不会有太大差别。
rust的Result实际上提供的是类型化，可转换，易于处理的错误处理方式。

### 实现demo
#### 客户端程序
``` rust src/bin/client.rs
use std::net::SocketAddr;
use std::str::FromStr;

use futures::SinkExt;
use futures::StreamExt;
use tokio::net::UdpSocket;
use tokio_udp;
use tokio_udp::Result;
use tokio_util::bytes::BufMut;
use tokio_util::bytes::BytesMut;
use tokio_util::codec::BytesCodec;
use tokio_util::udp::UdpFramed;

#[tokio::main]
async fn main() -> Result<()> {
    let socket = UdpSocket::bind("127.0.0.1:0").await?;
    let mut frame = UdpFramed::new(socket, BytesCodec::new());
    let addr = SocketAddr::from_str("127.0.0.1:8082")?;
    let mut bytes = BytesMut::with_capacity(64);
    bytes.put("Hello, world!".as_bytes());
    frame.send((bytes, addr)).await?;
    let next = frame.next().await;
    if let Some(Ok((bytes, addr))) = next {
        let str = String::from_utf8_lossy(&bytes[..]);
        println!("ret addr:{}, msg:{}", addr, str);
    }
    Ok(())
}
```

#### 服务端程序
``` rust src/bin/server.rs
use futures::SinkExt;
use futures::StreamExt;
use tokio::net::UdpSocket;
use tokio_udp;
use tokio_udp::Result;
use tokio_util::bytes::BufMut;
use tokio_util::bytes::BytesMut;
use tokio_util::codec::BytesCodec;
use tokio_util::udp::UdpFramed;

#[tokio::main]
async fn main() -> Result<()> {
    let socket = UdpSocket::bind("127.0.0.1:8082").await?;
    let mut frame = UdpFramed::new(socket, BytesCodec::new());
    loop {
        let next = frame.next().await;
        if let Some(Ok((bytes, addr))) = next {
            let str = String::from_utf8_lossy(&bytes[..]);
            if &str == "over" {
                break;
            }
            println!("addr:{}, msg:{}", addr, str);

            // 返回消息
            let mut bytes = BytesMut::with_capacity(64);
            bytes.put("ret".as_bytes());
            frame.send((bytes, addr)).await?;
            println!("ret");
        }
    }

    Ok(())
}
```

#### 短评
以前只写过tcp程序，现在也算是写过udp程序了。
经过这个demo的编写，我有以下收获:
了解到udp和tcp程序在写法上的不同之处；
了解到UdpFramed在工作时的一些行为；
了解到0端口是向操作系统申请一个随机的端口；
