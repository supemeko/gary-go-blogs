---
title: Flutter电商项目学习笔记 01-项目介绍
date: 2024-04-25 21:51:40
updated:
tags:
  - "文章"
  - "学习笔记"
categories: 
  - "学习笔记"
  - "Flutter电商项目"
description: "最近开始学习一个开源的电商项目，最开始只是想学习flutter，但是为了跑起flutter的电商项目，被迫学习了很多知识。
其中包括Flutter框架，golang语言开发的后端，采用go-zero微服务框架，又要学习gPRC, etcd等项目。
本来以为完事了，结果后台管理页面是react写的，还用了ant-design-pro。"
cover: https://jsd.012700.xyz/gh/jerryc127/CDN@latest/cover/default_bg.png
---

## 写在前面

### 起因
最近开始学习一个开源的电商项目Flutter Mall，最初只是想学习flutter，但是项目能够跑起来，被迫学习了很多知识。
其中包括Flutter框架，golang语言开发的后端，采用go-zero微服务框架，又要学习gPRC, etcd等项目。
本来以为完事了，结果后台管理页面是react写的，还用了ant-design-pro。
为了写博客还顺带学习了一下hexo，博客的主题是butterfly🤣

{% btn 'https://feihua.github.io/mall/app/overview.html',Flutter Mall,far fa-hand-point-right, outline blue  %}
{% btn 'https://butterfly.js.org/',Butterfly,far fa-hand-point-right, outline blue  %}

### 系列的主要内容
这个系列文章《Flutter电商项目学习笔记》的主要内容是在开源电商项目Flutter Mall的基础上进行二次开发，并将开发的过程记录下来。
我将通过完成项目中一些尚未完善的部分来验证所学的知识。文章里面会有一些讲解，但是教学不会是文章的主题。
第一章节在介绍项目的时候，我会给出一些我使用的教程的链接地址。里面有完整的教学视频可供参考。

### 叠个甲
博主是初学者，难免会出现一些错误，请读者见谅。

## 01-项目介绍
后续会在这个开源项目上做一些二次开发，那么首先就是要介绍这个开源项目的情况。

### 项目所使用的技术栈
- 移动端 Flutter Mall
    - 概述: 一个基于 Flutter 框架实现的电商系统移动端项目
    - Flutter: 使用 Flutter 框架进行跨平台移动应用开发
    - Dart: 使用 Dart 编程语言，与 Flutter 框架配套使用。
- 后端 Zero Admin
    - 概述: Flutter Mall 所使用的微服务后端
    - Go: 使用 Go 编程语言
    - go-zero: Go语言的微服务开发架构
    - etcd: 分布式键值存储系统，常用于服务发现、配置管理
    - gRPC & Protobuf: 谷歌的远程过程调用框架
    - makefile: 使用makefile管理构建和发布项目
- 后台管理页面 Zero Admin UI
    - 概述: 一个基于 React 实现的电商后台管理系统的前端项目
    - React: 使用现代化的前端框架，提高开发效率和用户体验。
    - Ant Design Pro: 阿里开源的中后台管理页面的解决方案。

`后端 Zero Admin`: https://github.com/feihua/zero-admin.git
`后台管理页面 Zero Admin UI`: https://github.com/feihua/zero-admin-ui.git
`移动端 Flutter Mall`: https://github.com/feihua/flutter_mall.git
`项目文档`: https://feihua.github.io/

可以看出这个项目的技术栈还是非常先进的。🙌
里面用到的技术没有过多的封装，基本上都是官方提供的方案直接拿来用，适合学习。😎

### 本人已掌握的技术栈
 - javascript/vue2开发Web页面
 - java spring cloud微服务开发

什么都不会就慢慢学呗。🥲

## 移动端Flutter Mall
`Flutter官方网站`: https://docs.flutter.dev/
`Flutter视频教程`: itying.com Flutter 入门实战教程

1. 跟着视频了解Flutter如何安装，学习基础组件的使用。😀
    通过视频学习的好处是能够快速了解开发流程，缺点就是还需要重新看一遍官方文档，避免教程视频遗漏知识点。
2. 使用Android Studio进行开发，配置环境让人头大。😡
    期间遇到自带的AVD模拟器无法使用网络的问题，结果重新配置了一台虚拟设备就行了。😟
3. Flutter框架
    - Flutter使用状态和渲染分离的设计，当状态改变时，页面随之改变，而框架会在内部避免不必要的渲染花销。
由于之前学习过Bevy原生UI和egui立即模式UI开发，理解这种设计的工作机制并不算困难。
立即模式下，UI界面会以每秒60次的频率对画面进行一次完整的渲染，其根据实时游戏状态对进行画面实时渲染。
立即模式适合简单画面、个性化的设计的UI设计需求。
而对于页面开发来说，页面开发的画面较为复杂，设计相对固定，这种情况下Flutter就更为适合了。
Flutter的Weiget的状态发生变化后，根据Weiget的状态重新渲染Weiget，这就避免了手动动态操作Weiget变化带来的复杂性。
不像立即模式UI每一帧都会重新渲染所有的Weiget，Flutter仅重新渲染发生改变的Weiget，这样做提高了性能。
但是Flutter不像游戏那样请求分配很高的性能，也可能是Flutter拥有更加复杂的Weigets，因此开发需要注意尽可能避免不必要的Weiget渲染（Weiget.build()）。

    - Flutter对程序员开放统一的Weiget接口，隐藏内部的实现细节，以此实现跨平台的能力。👍
4. Dart编程语言
    - 和javascript差不多，很容易学习。👌
    - 相比javascript有强类型系统，相比typescript，dart的类型系统更容易使用和阅读❤️
    - 并且同样有动态类型，编写起来非常方便💕

{% note blue simple %}
✍️登录接口，前后端定义的用户名好像不一致，一个是username，一个是account。
{% endnote %}

## 后端Zero Admin
`Go语言官方地址`: https://go.dev/
`go-zero git仓库`: https://github.com/zeromicro/go-zero
`B站视频《go-zero零基础入门教程|go微服务开发必学教程》`: https://www.bilibili.com/video/BV1kM411X7Cp?p=1
`B站视频《etcd原理与实践》`: https://www.bilibili.com/video/BV1fD4y1g75B/
`B站视频《【狂神说】gRPC最新超详细版教程通俗易懂 | Go语言全栈教程》`: https://www.bilibili.com/video/BV1S24y1U7Xp
`B站视频《Makefile 20分钟入门，简简单单，展示如何使用Makefile管理和编译C++代码》`:https://www.bilibili.com/video/BV188411L7d2

1. 项目使用go语言，中间遇到依赖下载不上来的情况，有点心累。😒😥
2. go语言比较简单，边用边学了。
3. go-zero使用谷歌的微服务生态，反正挺科学的。👍
4. 之前学习过java的微服务架构，再学go-zero并不困难。👌
5. etcd的客户端和redis类似，不过更多的承载一个服务发现和分布式状态存储的作用。
6. gRPC & Protobuf 谷歌的微服务使用Protobuf作为远程过程调用的数据传输格式，用gRPC生成对应调用的代码，负责调度。
7. go-zero通过各种工具生成代码，让开发可以专注业务开发。
8. 需要学会makefile的基础语法。没找到针对Go语言开发的教程。😓
9. 总的来说，go语言的Zero Admin后端还是挺清晰的。

{% note blue simple %}
🔔使用makefile启动脚本中使用了`copy`，但是在linux环境执行make build可能会报错。
{% endnote %}

## 后台管理界面 Zero Admin UI
`React`: https://react.dev/
`Ant-Design-Pro官方地址`: https://pro.ant.design/
`Ant-Design-Pro新手需知`: https://pro.ant.design/zh-CN/docs/introduction
`ProComponents`: https://procomponents.ant.design/
`B站视频《30分钟学会React18核心语法 可能是你学会React最好的机会 前端开发必会框架 无废话精品视频》`: https://www.bilibili.com/video/BV1pF411m7wV

![](/img/posts/studys-zero-mall/01-01-ant-design-pro.png "ANT DESIGN PRO")
1. 《新手需知》要一个个看，务必弄清楚每个组件的作用。
2. 【注意】我没看有关Ant-Design-Pro的B站教程，这应该是一个错误。
3. react和flutter一样，都是根据状态会对函数式组件进行重新渲染，内部有使用虚拟DOM进行优化，开发时也要注意尽量减少函数式组件的渲染。
不过react相比flutter更加原始，代码没有flutter清晰，类型系统也不容易阅读。😒

{% note blue simple %}
💡`npm run dev`在linux平台会可能报错，原因是代码中设置环境变量用到了`SET NODE_OPTIONS=--openssl-legacy-provider`，`SET`不是linux平台的命令。需要改成 `cross-env NODE_OPTIONS=--openssl-legacy-provider` 才能顺利启动项目。
{% endnote %}

## 写在后面
想要快速学习的话，个人建议先看B站教程或者那种培训课开发的课程。
这些教程的能够让你熟悉一个完整的开发流程，这点不是官方教程能够提供的。
如果你有几年开发经验，其实可以不用跟着教程步骤一步步走，2倍速看老师操作就好了，也不用动脑，就挺轻松的。
如果已经熟悉开发流程，跟着官方教程学是更好的方式，最终还是要看官方教程和文档的，不然会遗漏知识点。

下一章节，我会展示修复后台管理系统，不回显服务端业务异常的问题。
