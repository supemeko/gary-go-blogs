---
title: Flutter电商项目学习笔记 04-完善我的订单页面查询接口
date: 2024-05-17 00:32:46
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

## 介绍
### 概要
本章节对移动端APP中，完善我的订单功能。
该页面位于移动端APP {% label '我的 > 全部订单' blue %} ,改造前已经完成基础列表展示的功能，但是"待退货"、"待支付"等不同Tab显示完全相同的数据。
该章节主要内容是对这个页面进行改造，使得不同Tab需要显示不同的数据

## 正文
### 不同Tab显示不同的数据
#### 接口描述不满足前端使用
``` diff go-zero
type OrderListReq {
     Current int64 `path:"current,default=1"`
     PageSize int64 `path:"pageSize,default=5"`
-    Status int64 `path:"status"` // 订单状态：0->待付款；1->待发货；2->已发货；3->已完成；4->已关闭；5->无效订单
+    Status []int64 `json:"status"` // 订单状态：0->待付款；1->待发货；2->已发货；3->已完成；4->已关闭；5->无效订单
}

//按状态分页获取用户订单列表
@handler OrderList
-    get /orderList/:status/:current (OrderListReq) returns (OrderListResp)
+    get /orderList/:current (OrderListReq) returns (OrderListResp)
```

"全部"页面中应该包含五种订单状态的数据，而接口中只有一个字段，按照改造前设置的接口规范需要请求后端5次；
改造方案：
1. 修改接口描述，将"status"更改为"query type"，服务端和客户端之间传递，使用查询类型传递组合信息
    - 优点：改造简单
    - 缺点：
        - 不符合前后端分离的思想
        - 可能需要后端维护额外的`Map<QueryType, Status[]>`
        - 可能会导致QueryType和Status码值冲突;

2. 修改接口描述，使用"status"传递位掩码信息
    - 实现方案：
        - C语言中位域（Bit-fields）的高级玩法，8个案例代码告诉你怎么玩: https://zhuanlan.zhihu.com/p/636551863
        - 奇怪的知识——位掩码: https://zhuanlan.zhihu.com/p/352025616
    - 优点：接口清晰
    - 缺点：有一定门槛，需要开发了解位操作，需要修改字段名字，减少误解

3. 修改接口描述，使用"status[]"传递数据信息
    - 优点: 接口清晰，不需要修改字段名，不需要增加额外的概念
    - 缺点: 需要使用数组来传递信息

最后我选用了第三种方案

#### 修改过程
1. 修改 api 和 proto 文件
  - HTTP接口文件 `api/front/doc/api/order.api`
  - gRPC接口文件 `rpc/oms/oms.proto`
2. 使用goctl生成对应代码
  - 生成HTTP接口命令 `goctl api go -api ./api/front/doc/api/front.api -dir ./api/front/`
  - 生成gRPC接口命令 `goctl rpc protoc rpc/oms/oms.proto --go_out=./rpc/oms/ --go-grpc_out=./rpc/oms/ --zrpc_out=./rpc/oms/ -m`
3. 修复上诉命令执行完毕后的问题
  - SQL查询语句

#### mysql修改
``` go
if len(in.Status) > 0 {
  where = where + fmt.Sprintf(" AND status in (%s)", func() string {
    s := ""
    for i, v := range in.Status {
      if i > 0 {
        s += ", "
      }
      s += fmt.Sprintf("%d", v)
    }
    return s
  }())
}
```
有一种大道至简的美感😓
`in.Status`是`int[]`数据，虽然使用客户端输入数据进行拼接，但已经保证`in.Status`是整型数据，因此不存在SQL注入的风险

#### 其他事项
- 先保证git仓库整洁，然后执行goctl指令，执行完查看git diff，这样就知道代码生成工具是否执行正确了；
- goctl实际上有生成dart model的指令，goctl dart api *，但是这个项目没有使用这个命令；
- 之前只把`status`改成`int[]` 忘记把后面的`path:"status"`改成`json:"status"`，这个问题排查了很长时间；
