---
title: Flutter电商项目学习笔记 05-完善我的订单页面页面
date:  2024-06-10 23:16:16
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
在完成别的项目，不太有时间学习flutter和写学习笔记了；
以后学习笔记稍微写短一点，按小功能模块记录学习笔记了；

## 介绍
### 概要
本章节对移动端APP中，完善我的订单功能。
该页面位于移动端APP {% label '我的 > 全部订单' blue %}，先前的改造中，已经使得不同的Tab页可以查询不同的订单状态组合的页面。
现在对页面进行一些改造，现在的页面和京东长得不太一样，让他的逻辑更像京东一点；

### 改造前
{% gallery %}
![改造前](/img/posts/studys-zero-mall/05-01-image-before-memberOrderPage.png)
{% endgallery %}

### 改造后的页面
{% gallery %}
![改造前](/img/posts/studys-zero-mall/05-02-image-after-memberOrderPage.png)
{% endgallery %}

### 改造点
1. 单个商品时，显示商品图片和商品信息；多个商品时，仅显示商品图片；
2. 卡片的按钮数量、按钮内容需要按照订单信息进行个性化显示；
3. 卡片的按钮数量不超过4个时，依次显示按钮；卡片的按钮数量超过4个时，依次显示3个按钮，其余的按钮放入更多里面；

## 正文
### 基本页面改造
1. 使用**TabBarView**和**PageView**的选择
PageView的官方用例，比较符合订单页面的场景；同时TabBarView当然更是如此；😥😥😥
PageView相对底层，更容易控制，但是TabBarView语义上更符合场景，两种方案都可以满足当前场景的需要，花了一些时间，但是没有找到哪一种比另一种更适合的合理理由，总之最后选择了PageView。具体页面由自定义组件**TabWidget**，如何切换Tab和TabWidget的渲染并不耦合，因此可以随时将PageView改成TabBarView而不影响其他部分，当前选择PageView问题不大。
2. 使用PageView.builder优化TabWidget渲染
```dart 使用PageView.builder代替PageView的关键逻辑
PageView.builder(
  itemCount: _PageTabs.values.length,
  itemBuilder: (context, index) {
    final tab = _PageTabs.values[index];
    final tabData = _getTabData(tab);
    return _TabWidget(
      tab: tab,
      data: tabData,
    );
  },
)
```
相比原来的实现方式每次都渲染TabWidget[]的所有元素，现在只在有需要的时候渲染指定TabWidget
3. 使用FutureBuilder优化TabWidget渲染
```dart 使用FutureBuilder优化TabWidget渲染的关键逻辑
return FutureBuilder<List<OrderData>>(
  future: _queryTabData(tab),
  builder: (BuildContext context,
      AsyncSnapshot<List<OrderData>> snapshot) {
    if (snapshot.connectionState ==
        ConnectionState.waiting) {
      return const Center(
          child: CircularProgressIndicator());
    } else if (snapshot.hasError) {
      // 当Future发生错误时，显示错误提示的UI
      return Text('Error: ${snapshot.error}');
    } else {
      // 当Future成功完成时，显示数据
      return _TabWidget(
        tab: tab,
        data: snapshot.data!,
      );
    }
  },
);
```
如果当前页面没有数据时，请求queryTabData获取后端数据，同时向用户展示一个正在加载的动画；请求数据成功后，再对页面进行完整的渲染。
4. 结果
{% gallery %}
![结果](/img/posts/studys-zero-mall/05-03-image-structure-MemberOrderPage.png)
{% endgallery %}

### 单个商品和多个商品不同的展示效果
#### 展示效果
{% gallery %}
![单个及多个商品时不同的展示效果](/img/posts/studys-zero-mall/05-04-image-single-or-multi-goods-on-ordercard.png)
{% endgallery %}
仅包含一个商品时，展示商品图片+商品文字信息
包含多个商品时，展示每个商品的图片信息

#### 声明式定义ui组件
``` dart 单个商品及多个商品分别渲染OrderCard的处理
class _OrderCard extends StatelessWidget {
  final OrderData orderData;
  final Function(String name, dynamic params) orderEvent;

  const _OrderCard(
      {super.key, required this.orderData, required this.orderEvent});

  @override
  Widget build(BuildContext context) {
    // ...
    return Card(
        margin: const EdgeInsets.all(10),
        child: Container(
          padding: const EdgeInsets.only(left: 15, right: 15),
          child: Column(
            children: [
              SizedBox( ... ), // 时间及商品状态
              SizedBox(
                  height: 100,
                  child: Row(
                    crossAxisAlignment: CrossAxisAlignment.center,
                    children: [
                      ...orderData.orderItemList.length > 1
                          ? middleLayout1()
                          : middleLayout0(),
                      SizedBox( ... ) // 价格信息
                    ],
                  )),
              ListTile( ... ) //按钮信息
            ],
          ),
        ));
  }

  // 单个商品时渲染流程
  List<Widget> middleLayout0() { ... }

  // 多个商品时渲染流程
  List<Widget> middleLayout1() { ... }
}
```
使用OrderCard类定义"订单卡片"，根据状态渲染不同的内容

#### 单个商品时的渲染流程
``` dart 单个商品时的渲染流程
class _OrderCard extends StatelessWidget {
  final OrderData orderData;
  final Function(String name, dynamic params) orderEvent;

  const _OrderCard(
      {super.key, required this.orderData, required this.orderEvent});

  @override
  Widget build(BuildContext context) { ... }

  // 单个商品时渲染流程
  List<Widget> middleLayout0() { 
    // 商品描述，如果只有1件商品的话，就需要显示商品信息
    final firstItem = orderData.orderItemList[0];

    final firstItemDesc = () {
      // 最多显示多少个UTF16编码单元（可以显示的文字数量是有限的，没必要显示太多）
      const maximumNumberOfUnitCodesAtDisplay = 60;
      var desc = firstItem.productName;
      var attrList = jsonDecode(firstItem.productAttr);
      for (var attr in attrList) {
        desc += attr['key'] + attr['value'];
        if (desc.length > maximumNumberOfUnitCodesAtDisplay) {
          break;
        }
      }
      return desc;
    }();

    return [
      CachedImageWidget(
        96,
        96,
        firstItem.productPic,
        fit: BoxFit.contain,
      ),
      const SizedBox(width: 1),
      Expanded(
        child: ListTile(
          title: Text(
            firstItemDesc,
            maxLines: 2,
            overflow: TextOverflow.ellipsis,
          ),
        ),
      )
    ];
  }

  // 多个商品时渲染流程
  List<Widget> middleLayout1() { ... }
}
```
😎小技巧： 使用 `final value = (){ return ret; }()` 作为块表达式【块表达式各种好用，没有块表达式各种难受😥】
取订单信息中第一个商品（唯一一件商品）的商品图片信息和商品文字信息。
商品文字信息需要拼接商品名称和商品属性。👌虽然不是很必要，手动设置了仅展示约60个字符的操作（实际是60个UTF16编码单元）

#### 多个商品时的渲染流程
```
class _OrderCard extends StatelessWidget {
  final OrderData orderData;
  final Function(String name, dynamic params) orderEvent;

  const _OrderCard(
      {super.key, required this.orderData, required this.orderEvent});

  @override
  Widget build(BuildContext context) { ... }

  // 单个商品时渲染流程
  List<Widget> middleLayout0() { ... }

  // 多个商品时渲染流程
  List<Widget> middleLayout1() { 
    var orderItemList = orderData.orderItemList;
    // 最多允许显示多少件商品图片（可以显示的图片数量是有限的，没必要显示太多）
    const maximumNumberOfPicturesAtDisplay = 4;
    final pictureIterator = orderItemList
        .sublist(0, min(orderItemList.length, maximumNumberOfPicturesAtDisplay))
        .map((elem) => elem.productPic);

    return [
      Expanded(
        child: SingleChildScrollView(
          scrollDirection: Axis.horizontal,
          child: Row(
            children: pictureIterator
                .map((elem) => Row(children: [
                      CachedImageWidget(
                        96,
                        96,
                        elem,
                        fit: BoxFit.contain,
                      ),
                      const SizedBox(width: 8)
                    ]))
                .toList(),
          ),
        ),
      )
    ];
  }
}
```
使用一些子表和函数式语法限制图片至多显示4个
```
final pictureIterator = orderItemList
    .sublist(0, min(orderItemList.length, maximumNumberOfPicturesAtDisplay))
    .map((elem) => elem.productPic);
```
### 卡片按钮信息处理
#### 使用mixin定义数据处理
``` dart
mixin _PageData {
  List<List<OrderData>?> list = [..._PageTabs.values.map((e) => null)];

  Future<List<OrderData>> _queryTabData(_PageTabs tab) async {
    const page = 1;
    Response result = await HttpUtil.get("$orderListDataUrl$page", data: {
      "status": tab.orderStatuses,
    });
    list[tab.index] = OrderPageResp.fromJson(result.data).data;
    return list[tab.index]!;
  }

  Future<void> _cancelOrder(int orderId) async {
    Response result = await HttpUtil.get("$orderCancelUrl/$orderId");
    developer.log('$result');
  }

  Future<void> _handleEvent(
      OrderData orderData, String name, dynamic value) async {
    switch (name) {
      case "取消":
        _cancelOrder(orderData.id);
        return;
    }
    developer.log("${orderData.id} $name $value");
  }

  List<OrderData>? _getTabData(_PageTabs tab) {
    return list[tab.index];
  }
}
```
#### 组件树状结构
***_MemberOrderPageWidgetState with _PageData -> FutureBuilder -> _TabWidget[] -> _OrderCard[] -> _OrderCardAction[]***
#### 各组件分别进行哪些操作
1. `_PageData` 中定义了组件的数据行为，为`_MemberOrderPageWidget`提供与后端交互的能力
2. `FutureBuilder` 会通过_PageData._queryTabData查询指定Tab中所有的订单列表信息
3. `_TabWidget` 设置_PageData._handleEvent作为其回调函数
4. `_TabWidget` 中包含 `final List<OrderData> data` 信息，根据data[index]可生成指定_OrderCard，同时为_OrderCard指定回调函数orderEvent
``` dart
orderEvent: (name, params) async {
  await handleEvent(data[index], name, params); //注意handleEvent将指向_PageData._handleEvent函数
},
```
此时_PageData._handleEvent的第一个参数“订单信息”已经被设置为data[index]。
5. `_OrderCard` 在build阶段，一次性生成所有按钮的回调动作_OrderCardAction[]
``` dart
class _OrderCard extends StatelessWidget {
  final OrderData orderData;
  final Function(String name, dynamic params) orderEvent;

  const _OrderCard(
      {super.key, required this.orderData, required this.orderEvent});

  @override
  Widget build(BuildContext context) {
    // ...
    final actions = _OrderCardAction.actions(orderData, orderEvent);

    // ...
    return Card( ... );
  }
  // ...
}
```
6. `_OrderCardAction`根据订单数据信息和组件回调函数，遍历每一种类型的按钮为其生成对应的_OrderCardAction；
``` dart
enum _OrderCardActionType {
  buyAgain("再次购买", defaultMatch),
  afterSales("退换/售后", afterSalesMatch),
  sell("卖了换钱", sellMatch),
  invoice("查看发票", defaultMatch),
  evaluate("评价晒单", defaultMatch2),
  delete("删除", deleteMatch),
  cancel("取消", cancelMatch),
  ;

  final String title;
  final (bool, dynamic) Function(OrderData orderData) condition;

  const _OrderCardActionType(this.title, this.condition);
}

class _OrderCardAction {
  final _OrderCardActionType actionType;
  final Function() doAction;

  const _OrderCardAction({required this.actionType, required this.doAction});

  static List<_OrderCardAction> actions(OrderData orderData, Function(String name, dynamic params) orderEvent) {
    return _OrderCardActionType.values
        .map((actionType) {
          var (cond, dyn) = actionType.condition(orderData);
          return (cond, _OrderCardAction(actionType: actionType, doAction: () => {
            orderEvent(actionType.title, dyn)
          }));
        })
        .where((value) => value.$1)
        .map((e) => e.$2)
        .toList();
  }
}
```
通过设置按钮类型的condition函数，可以决定该订单是否需要该类型的按钮
🙌由按钮类型决定订单是否需要渲染自己，而不是由订单为自己依次添加按钮，非常合理👍
按钮类型在决定是否渲染自己的同时，返回回调函数所需要的额外信息，这样_PageData不再需要关心每种按钮的具体行为，这样非常好👌

7. `_OrderCardAction`提供按钮的类型信息和回调方法，使得`_OrderCard`可以利用这些信息对按钮进行渲染
比方说按钮的标题是`_OrderCardAction.actionType.title`

### 卡片按钮的展示效果
{% gallery %}
![结果](/img/posts/studys-zero-mall/05-05-image-actions-on-ordercard.png)
{% endgallery %}
从右往左放置按钮，如果按钮数量小于或等于四个，依次将其作为行按钮放置；
如果超过四个按钮，则将超过的部分和第四个按钮放置到"更多"中，然后显示前三个行按钮

#### 行按钮的渲染
``` dart
class _OrderCard extends StatelessWidget {
  final OrderData orderData;
  final Function(String name, dynamic params) orderEvent;

  const _OrderCard(
      {super.key, required this.orderData, required this.orderEvent});

  @override
  Widget build(BuildContext context) {
    final itemList = orderData.orderItemList;
    final actions = _OrderCardAction.actions(orderData, orderEvent);

    assert(itemList.isNotEmpty,
        'The order should have included at least one item, but order(${orderData.id}) did not');

    bool needMoreAction = actions.length > 4;
    int displayActionCount = needMoreAction ? 3 : min(actions.length, 4);
    return Card(
        margin: const EdgeInsets.all(10),
        child: Container(
          padding: const EdgeInsets.only(left: 15, right: 15),
          child: Column(
            children: [
              // ...
              ListTile(
                title: Row(
                  children: [
                    // ...
                    ...List.generate(displayActionCount,
                            (index) => displayActionCount - 1 - index)
                        .where((element) => element < actions.length)
                        .map((element) => actions[element])
                        .map((action) {
                          return <Widget>[
                            TextButton(
                              onPressed: () async {
                                await action.doAction();
                              },
                              style: TextButton.styleFrom(
                                  visualDensity: VisualDensity.compact,
                                  shape: const RoundedRectangleBorder(
                                      borderRadius:
                                          BorderRadius.all(Radius.circular(20)),
                                      side: BorderSide(
                                          color: Color(0xffaaacb0), width: 1))),
                              child: Text(action.actionType.title,
                                  style: const TextStyle(
                                      fontSize: 14, color: Color(0xff303133))),
                            ),
                            const SizedBox(width: 5)
                          ];
                        })
                        .expand<Widget>((element) => element)
                        .toList()
                  ],
                ),
              )
            ],
          ),
        ));
  }
  // ...
}
```
使用List.generate生成一个指定数量的倒序数组，然后通过索引获取对应的_OrderCardAction，然后将其渲染出来

#### 渲染更多按钮的条件
``` dart
class _OrderCard extends StatelessWidget {
  final OrderData orderData;
  final Function(String name, dynamic params) orderEvent;

  const _OrderCard(
      {super.key, required this.orderData, required this.orderEvent});

  @override
  Widget build(BuildContext context) {
    // ...
    return Card(
        margin: const EdgeInsets.all(10),
        child: Container(
          padding: const EdgeInsets.only(left: 15, right: 15),
          child: Column(
            children: [
              // ...
              ListTile(
                title: Row(
                  children: [
                    if (needMoreAction) ...[
                      SizedBox(
                        width: 30,
                        height: 48,
                        child: _MoreButtonWidget(
                            actions.skip(displayActionCount).toList()),
                      )
                    ],
                    // ...
                  ],
                ),
              )
            ],
          ),
        ));
  }
  // ...
}
```
需要更多按钮时【按钮数量超过4个】,渲染_MoreButtonWidget

#### 更多按钮的渲染
``` dart
class _MoreButtonWidget extends StatefulWidget {
  final List<_OrderCardAction> actions;

  const _MoreButtonWidget(this.actions, {super.key});

  @override
  State<StatefulWidget> createState() => _MoreButtonWidgetState();
}

class _MoreButtonWidgetState extends State<_MoreButtonWidget> {
  late List<_OrderCardAction> actions;
  final OverlayPortalController _tooltipController = OverlayPortalController();

  @override
  void initState() {
    super.initState();
    actions = widget.actions;
  }

  @override
  Widget build(BuildContext context) {
    final link = LayerLink();
    return TextButton(
      onPressed: () async {
        _tooltipController.toggle();
      },
      style: TextButton.styleFrom(
        padding: const EdgeInsets.all(0),
        visualDensity: VisualDensity.compact,
        shape: const RoundedRectangleBorder(
          borderRadius: BorderRadius.zero,
        ),
      ),
      child: OverlayPortal(
        controller: _tooltipController,
        overlayChildBuilder: (BuildContext context) {
          return Positioned(
            width: 100,
            child: CompositedTransformFollower(
              targetAnchor: Alignment.topRight,
              followerAnchor: Alignment.bottomLeft,
              offset: const Offset(-10, 0),
              link: link,
              child: Container(
                color: Colors.white,
                child: TapRegion(
                  consumeOutsideTaps: true,
                  onTapOutside: (event) async {
                    _tooltipController.hide();
                  },
                  child: Column(
                    children: actions.map(
                      (action) {
                        return SizedBox(
                          width: 100,
                          child: TextButton(
                            onPressed: () async {
                              await action.doAction();
                              _tooltipController.hide();
                            },
                            style: TextButton.styleFrom(
                              padding: const EdgeInsets.all(0),
                              shape: const RoundedRectangleBorder(
                                borderRadius: BorderRadius.zero,
                              ),
                            ),
                            child: Text(action.actionType.title),
                          ),
                        );
                      },
                    ).toList(),
                  ),
                ),
              ),
            ),
          );
        },
        child: CompositedTransformTarget(
            link: link,
            child: const Text('更多', style: TextStyle(fontSize: 14))),
      ),
    );
  }
}
```
需要配合使用OverlayPortal，CompositedTransformFollower，CompositedTransformTarget才能够让按钮的列表显示在更多按钮的右上方
😥翻了半天标准库源代码才知道要用这个组合。
🚕相比vue这样的框架，自己动手解决这样的问题会更容易。
