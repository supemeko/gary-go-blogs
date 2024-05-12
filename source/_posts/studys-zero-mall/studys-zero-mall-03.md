---
title: Flutter电商项目学习笔记 03-完善收货地址页面
date: 2024-05-03 22:08:17
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
本章节对移动端APP中，将完善收货地址的交互的功能。
该页面位于移动端APP { % label 我的 > 地址管理 blue% }，目前已经完成列表展示、地址详情展示的功能。但是无法进行新建和修改地址信息的操作
该章节主要内容包括：
1. 页面交互逻辑以及相应后端改造。
2. 获取设备信息，调用第三方接口优化用户体验。
3. 调用库实现从文本中提取地址信息功能。
4. 总结及可能进一步完善的地方

### 其他信息
收货地址后管页面位于 {% label '会员管理 > 会员列表' blue %}，点击查询{% label '会员地址' blue %}
会员收货地址表的表名 **ums_member_receive_address**

## 正文
### 页面交互逻辑以及相应后端改造
#### 后端接口
``` go 涉及的后端接口
MemberReceiveAddressAdd(context.Context, *MemberReceiveAddressAddReq) (*MemberReceiveAddressAddResp, error)
MemberReceiveAddressList(context.Context, *MemberReceiveAddressListReq) (*MemberReceiveAddressListResp, error)
MemberReceiveAddressUpdate(context.Context, *MemberReceiveAddressUpdateReq) (*MemberReceiveAddressUpdateResp, error)
MemberReceiveAddressDelete(context.Context, *MemberReceiveAddressDeleteReq) (*MemberReceiveAddressDeleteResp, error)
```
目前后端接口中已经包括Add,List,Update,Delete等方法。
其中Add, Update接口需要改造，其他接口不需要变动，这四个接口已满足移动端使用。

#### Add, Update接口改造
新增和修改收货地址接口（Add, Update接口）中，需要处理『默认地址』的相关逻辑，确保每个会员只有一个默认收货地址。
『是否为默认地址』和其他字段存放在会员收货地址表中，而Add和Update接口在将某条记录设置为『默认地址』前，会先进行移除原默认地址的操作。
- 移除原默认地址操作，涉及两条**SQL**语句
  1. 查询当前用户所有的收货地址的**query**语句；
  2. 当收货地址是『默认地址』时，通过**update**语句将其设置为『非默认地址』。
- 移除原默认地址操作，适当改造后仅需要一条**SQL**语句
  - `update ums_member_receive_address set default_status = 0 where member_id = ? and default_status = 1`

##### 新增数据库操作接口
``` diff
UmsMemberReceiveAddressModel interface {
	UmsMemberReceiveAddressModel interface {
    umsMemberReceiveAddressModel
    Count(ctx context.Context, in *umsclient.MemberReceiveAddressListReq) (int64, error)
    FindAll(ctx context.Context, in *umsclient.MemberReceiveAddressListReq) (*[]UmsMemberReceiveAddress, error)
    DeleteByIdAndMemberId(ctx context.Context, id int64, MemberId int64) error
    FindByIdAndMemberId(ctx context.Context, id int64, memberId int64) (*UmsMemberReceiveAddress, error)
+   CancelDefaultAddressByMemberId(ctx context.Context, memberId int64) error
  }
}
```
为**UmsMemberReceiveAddressModel**中新增一个接口。这里的**Model**对象类似**Spring**中的**Dao**对象。

``` go
func (m *customUmsMemberReceiveAddressModel) CancelDefaultAddressByMemberId(ctx context.Context, memberId int64) error {
	query := fmt.Sprintf("update %s set default_status = 0 where member_id = ? and default_status = 1", m.table)
	_, err := m.conn.ExecCtx(ctx, query, memberId)
	return err
}
```
实现新添加的移除原默认地址接口；
处理逻辑是根据**会员id**，将**default_status = 1**的所有记录修改为**default_status = 0**

##### 更新后端接口的实现
``` diff
    if in.DefaultStatus == 1 {
-           //查询会员所有有地址
-           memberList, _ := l.svcCtx.UmsMemberReceiveAddressModel.FindAll(l.ctx, &umsclient.MemberReceiveAddressListReq{
-               Current:  1,
-               PageSize: 100,
-               MemberId: in.MemberId,
-           })
+           err := l.svcCtx.UmsMemberReceiveAddressModel.CancelDefaultAddressByMemberId(l.ctx, in.MemberId)
 
-           for _, address := range *memberList {
-               //判断是否为默认,如果是,则修改
-               if address.DefaultStatus == 1 {
-                   address.DefaultStatus = 0
-                   _ = l.svcCtx.UmsMemberReceiveAddressModel.Update(l.ctx, &address)
-               }
+           if err != nil {
+               return nil, err 
+           }
    }
```
修改后端接口实现（新增收货地址和修改收货地址）；
这个接口仅在需要设置默认地址时，才会调用清除原默认地址的操作。

#### 页面改造

目前页面只具备展示信息的功能，表单数据不能进行修改，对应按钮没有调用后端接口。

<img src="/img/posts/studys-zero-mall/03-02-image-detail.png" width = "40%" height = "40%" alt="详情改造前"/>


##### 表单文本项
表单中的文字，目前使用Text展示，而Text没有文本框功能

``` tsx
Text("我是一段普通文本");

_name.text = "我是一个文本框"
TextField(
  controller: _name
  onChanged: (text) {
    setState(() {
      _name.text = text;
    });
  },
  inputFormatters: [
    LengthLimitingTextInputFormatter(20),
  ],
);

_name.text = "我是一个表单项"
TextFormField(
  validator: (value) => (value == null || value.isEmpty) ? "不能为空" : null,
  inputFormatters: [
    LengthLimitingTextInputFormatter(10),
  ],
  controller: _name,
  onChanged: (text) {
    setState(() {
      _name.text = text;
    });
  },
  style: textStyle),
);
```
- **Text**小部件只支持显示文字功能。
- **TextField**允许用户输入或更改文字信息。
- **TextFormField**提交表单时会校验提示信息是否正确，
- **controller**参数接收一个对象，管理文本
- **onChanged**当文本改变时，调用这个方法
- **setState**该函数被调用后会重新渲染页面
- **validator**设置表单校验函数，返回值是错误提示，校验成功时返回**null**

##### 设置表单
**TextField**需要被**Form**小部件包裹。
为**Form**指定一个key
```dart
class _AddressAddState extends State<AddressAdd> {
  final _formKey = GlobalKey<FormState>();
  // ...其他代码
}
```
指定Form小部件的key
``` dart
Form(
  key: _formKey,
  child: (...)
)
```

##### 表单提交
为按钮注册回调函数
``` dart
onTap: () async {
  if (_formKey.currentState!.validate()) {
    final url = ... ;
    var data = { ... };

    try {
      await HttpUtil.post(url, data: data);
      if (!context.mounted) return;
      Navigator.of(context).pop({'success': true});
    } catch (e) {
      print("失败:${e}");
    }
  }
},
```
- `_formKey.currentState!.validate()`返回Form中所有TextFromField的校验结果，校验通过后发送**后端请求**，否则展示错误信息

##### 跨页面传值
###### 静态路由传值到子页面
通过静态路由传值，将列表中的地址信息赋值给**data**传入**AddressAdd**小部件的构造函数
``` dart
Navigator.of(context).push(
  MaterialPageRoute(
    builder: (context) =>
      AddressAdd(data: data),
  ),
);
```

###### 有状态组件传值
先将**data**保存到`StatefulWidget`的成员变量中
``` dart
class AddressAdd extends StatefulWidget {
  final AddressListData? data;

  const AddressAdd({super.key, this.data});

  @override
  State<AddressAdd> createState() => _AddressAddState();
}
```

###### 初始化组件状态
``` dart
class _AddressAddState extends State<AddressAdd> {
  late int _defaultStatus;
  late bool _create;

  int? _id;
  final _name = TextEditingController();
  final _phoneNumber = TextEditingController();
  final _postCode = TextEditingController();
  final _province = TextEditingController();
  final _city = TextEditingController();
  final _region = TextEditingController();
  final _detailAddress = TextEditingController();
  final _formKey = GlobalKey<FormState>();

  @override
  void initState() {
    super.initState();
    final data = widget.data;
    _create = data?.id == null;
    _id = data?.id;
    _name.text = data?.name ?? '';
    _phoneNumber.text = data?.phoneNumber ?? '';
    _defaultStatus = data?.defaultStatus ?? 0;
    _postCode.text = data?.postCode ?? '';
    _province.text = data?.province ?? '';
    _city.text = data?.city ?? '';
    _region.text = data?.region ?? '';
    _detailAddress.text = data?.detailAddress ?? '';
  }
  
  //...其他代码
}
```
在**initState**方法中可以通过成员变量**widget**访问对应的**StatefulWidget**,通过**widget.data**访问传入子页面的收货地址数据。
- **_defaultStatus**: 记录是否为默认收货地址，新增页面为0
- **_id**: 记录的唯一索引，通过数据库自增索引获取，新增页面无此项
- **_create**: 记录是新建页面还是修改页面，通过**_id**是否有值判断

##### 标题及删除按钮
``` dart
appBar: AppBar(
  backgroundColor: Colors.white,
  title: Text(_create ? "新增收货地址" : "修改收货地址",
      style: const TextStyle(fontSize: 16, color: Colors.black)),
  centerTitle: true,
  actions: [ 
    if (!_create)
      IconButton(
        icon: const Icon(Icons.delete),
        onPressed: () async {
          // ...
        },
      ),
  ],
),
```

##### 替换颜色代码
将代码中的`Color(int.parse('fa436a', radix: 16)).withAlpha(255)`
替换为`Color(0xfffa436a)`
使用正则表达式：
- 搜索: `Color\(int\.parse\('([^']+)', radix: 16\)\)\.withAlpha\((255)\)`
- 替换为: `Color(0xff$1)`
##### 路由返回
``` dart
if (!context.mounted) return;
Navigator.of(context).pop();
```
通过`Navigator.of(context).pop()`返回上一个页面
使用**context**前需要判断**context.mounted**为true

##### 效果
{% gallery %}
![列表改造前](/img/posts/studys-zero-mall/03-01-image-list.png)
![详情改造前](/img/posts/studys-zero-mall/03-03-image-list-after.png)
![列表改造后](/img/posts/studys-zero-mall/03-02-image-detail.png)
![详情改造后](/img/posts/studys-zero-mall/03-04-image-detail-after.png)
{% endgallery %}

### 所在区域改造
#### 改造内容
目前为止，填写省市县的方式是用户手动输入，这并不合理。

#### 库的选择
本来想仿造京东，但苦于没有找到对应的库，决定自己写。写着写着发现原来有库，是我找库的姿势不对。😥

city_pickers提供三种样式: https://pub.dev/packages/city_pickers
flutter_city_picker样式仿照京东: https://pub.dev/packages/flutter_city_picker

以上两个库的接口设计，都是让人觉得疑惑。
最终决定，自己写数据接口的部分，样式拿flutter_city_picker的来用。

#### 接口设计
##### 命名
“所在区域”这个表单项的本质是获知收货地址属于哪个区域。
因此决定命名为**PostalPicker**，意为“邮递区域选择器”;

##### 警告
{% note danger simple %}
这部分设计只是我设想中的接口应该长什么样子
仅进行了粗糙的设计，没有经过验证，不应该用于实际产品🐱
{% endnote %}

##### 采用数据驱动方式
接口采用数据驱动的方式，大致分为三个部分：
 - Tables(DataSource): 使用简单的模型来存储和使用原始数据，其他原始数据格式统一转换为这种模型；
 - Page(Model): 页面统一使用的数据抽象模型；
 - Widget(View): 页面布局；

##### 目录结构及其用处
``` tree
postal_picker/
├── meta
│   └── static_data.dart   StaticData, 保存在App中的默认静态数据。
├── model
│   └── page.dart          Page，Widget所使用的统一数据抽象模型；
├── postal_picker.dart     Picker，用于弹出选择器窗口
├── table
│   ├── table_node.dart    Node表，决定邮递区域之间的关系
│   ├── table_postal.dart  Postal表，定义邮递区域
│   └── tables.dart        tables，存储所有Node表和Postal表的对象
└── view
    ├── postal_list_widget.dart      邮递区域选择列表
    ├── postal_picker_widget.dart    选择器的Widget
    └── third_listview_section.dart  用于邮递区域选择列表的第三方ListView实现
```

#### 表及数据
##### Node Table
``` dart
/// 用于描述postalCode的关系
class Node {
  /// 当前节点id
  final int id;

  /// 父节点id
  final int prevId;

  /// 当前节点级别
  final int lvl;

  /// 邮递区号
  final String postalCode;

  Node(this.id, this.prevId, this.lvl, this.postalCode);

  get isEmpty {
    return this == emptyNode;
  }

  get isNotEmpty {
    return !isEmpty;
  }
}

final emptyNode = Node(-1, -1, -1, "EmptyPostal");
final rootNode = Node(-2, -1, 0, "RootPostal");
```
Node表 被认为是用来描述PostalCode树形关系的表

##### Postal Table
``` dart
/// 邮递区域
class Postal {
  /// 该邮递区域的邮递区号
  final String postalCode;
  /// 该邮递区域的名字
  final String name;
  /// 该邮递区域名字的拼音首字母
  final String firstLetter;

  Postal(
      {required this.postalCode,
      required this.name,
      required this.firstLetter});

  bool get isEmpty {
    return this == emptyPostal;
  }

  bool get isNotEmpty {
    return this != emptyPostal;
  }
}

final emptyPostal = Postal(
  postalCode: "EmptyPostal",
  name: "待选择",
  firstLetter: "-",
);

final rootPostal = Postal(
  postalCode: "RootPostal",
  name: "根节点",
  firstLetter: "-",
);
```
Postal表 被认为是一个包含几个简单字段用来描述邮递区域的表
注意，默认使用的`postalCode`不是中国邮政系统中所使用的"邮政编码（Postal Code）"，而是中国地区编码

##### Tables
``` dart
class Tables {
  /// 节点表 & id索引
  /// 应该将其视为`Node[]`, 并且`Node[]`附带一个`id`索引
  Map<int, Node> nodes = {};

  /// 邮递区域表 & 邮递区域邮递区号索引
  /// 应该将其视为`Postal[]`, 并且`Postal[]`附带一个`postalCode`索引
  Map<String, Postal> postalList = {};

  /// 通过邮递区号快速搜索节点
  final Map<String, Node> cachedNodes = {};

  Tables();

  /// 获取邮递区域在表中的节点
  Node getNodeByPostalCode(String postalCode) { ... }

  /// 获取邮递区域的子区域列表
  List<Postal> getPostalListByPostalCode(String postalCode) { ... }

  /// 获取邮递区号对应的邮递区域
  Postal? getPostalByPostalCode(String postalCode) { ... }

  /// 上一个node
  Node getFatherNode(Node node) { ... }
}

/// 返回静态数据作为Tables
Tables defaultTables() { ... }
```
Tables对象，将Node和Postal表数据存放到这个对象中，提供一些访问数据的方法；
本质上来说提供Tables提供访问Node[]和Postal[]的能力。

#### Page对象
##### 类定义
``` dart
/// 页面数据
class Page {
  /// 存放着所有邮递区域相关的数据
  final Tables tables;

  /// 最多几级规划。【省市县乡，则4级；如果已经确定省份，选择市县乡，则3级；】
  final int numberOfTabs;

  /// 最高级别
  int get maxLevel {
    return numberOfTabs + 1;
  }

  /// 第n级选择的邮递区域对应的子集合区域【0：根目录下的选项，1：广东省下的选项，2：深圳市下的选项，3：福田区下的选项，4:福田街道下面的选项(无法选择)】
  List<List<Postal>> activityList = [];

  /// 第n级选择的邮递区域【0：根节点，1：广东省，2：深圳市，3：福田区，4：福田街道】
  List<Postal> selectiveList = [];

  /// 提交页面的回调函数 callback(this)
  void Function(Page page) callback;

  Page(
      {required this.tables,
      required this.numberOfTabs,
      required this.callback}) {
    for (int i = 0; i < maxLevel + 1; i++) {
      selectiveList.add(emptyPostal);
      activityList.add([]);
    }
  }

  /// 选择某个邮递区域，并查找该邮递区域的子节点
  bool pick(Postal postal) { ... }
  /// 可选tab的数量
  int get numberOfTabsIfOptional { ... }
  /// 获取所有tab的标题
  Iterable<(int, String)> get tabTitles { ... }
  /// 获取所有tab的已选项和备选列表
  Iterable<(int, Postal, List<Postal>)> get tabOptions { ... }
}
```
##### Pick Postal 选择邮递区域
当用户在页面种选择某个区域时，pick方法`bool pick(Postal postal)`将被调用；
如用户选择深圳市440300时（440300是深圳市的地区编号），页面会调用`Page.pick(深圳市)`，此时Page会相应的更新成员变量*activityList*和*selectiveList*为以下值：
- *selectiveList* [ 全国根节点, 广东省, 深圳市, 未选择的节点(区级), 未选择的节点(街道级) ]
- *activityList* [ 34个省份, 21个广东省的地级市, 9个深圳市的市辖区, 空(n个街道，未选择区级时为空), 空(街道下面无规划，恒为空) ]

而Widget只需要负责根据*selectiveList*，*activityList*绘制画面。

#### 使用Page数据
调用方设置Page时，会传递一个`callback(Page page)`，回调函数中使用*selectiveList*中的数据即可
``` dart
final page = postal_page.Page(
  tables: tables,
  numberOfTabs: 4,
  callback: (page) => {
    setState(() {
      for (int i = 0; i < 4; i++) {
        // for 1..=4
        final pageIndex = i + 1;
        final postal = page.selectiveList[pageIndex];
        postalList[i] = postal.isNotEmpty ? postal.name : '';
        if (postal.isNotEmpty) {
          _postalCode = postal.postalCode;
        }
      }
      _postalRegion.text = _postalRegionDesc;
    })
  },
);
```

#### 字母索引的分组和排序
##### 基本思路
上面提到*activityList*中存放了每个Tab的`List<Postal>`数据，这个数据是未分组和排序的；
`pick(Postal postal)`方法，会根据*tables*中的数据，重新考察选中的*Postal*与其他*Postal*的关系，并最终重新刷新*activityList*，因此不需要对activityList[]以及activityList[][]进行排序。数据进入邮递区域列表组件*PostalListWidget*时，再对其进行分组和排序即可；

🔔*PostalListWidget*对应部分如下图所示
{% gallery %}
![改造后](/img/posts/studys-zero-mall/03-07-image-whereis-postallistwidget.png)
{% endgallery %}

##### PostalListWidget中分组和排序的代码实现
``` dart
class PostalListWidgetState extends State<PostalListWidget>
    with AutomaticKeepAliveClientMixin {

  /// 邮递区域选项
  List<Postal> _options = [];

  /// 邮递区域选项首字母分组
  List<_PostalListGroupByFirstLetter> _optionsGroupByFirstLetter = [];

  void _calculateOptionsGroupBy() {
    _options = widget.options ?? [];
    _optionsGroupByFirstLetter = groupBy(_options, (p0) => p0.firstLetter)
        .entries
        .map((e) => _PostalListGroupByFirstLetter(
            firstLetter: e.key, postalList: e.value))
        .sorted((a, b) => a.firstLetter.compareTo(b.firstLetter))
        .toList();
  }

  // ...其他代码
}
```
- 其中*groupBy*是标准库`dart:collection`中的方法;
- 实现步骤
1. 首先将`activityList[TabIndex]`传递给**PostalListWidget**的`_options`字段
2. 然后根据首字母,对`_options`进行分组和排序,并将结果赋值给`_optionsGroupByFirstLetter`
4. 第三方组件*ExpandableListView*负责对`_optionsGroupByFirstLetter`进行渲染得到想要的效果

#### 点击字母列表滑动
```dart
  void clickIndexBar(int index) {
    if (index == 0) {
      _scrollController.jumpTo(0);
      return;
    }

    final groupIndex = index;
    final itemIndex = _optionsGroupByFirstLetter.sublist(0, index).fold(0,
        (previousValue, element) => previousValue + element.postalList.length);
    final position =
        widget.itemHeadHeight! * groupIndex + widget.itemHeight! * itemIndex;
    _scrollController.jumpTo(position);
  }
```
点击右侧字母时，列表需要对应进行滑动，以上是实现的代码；
点击字母后，以上方法将被调用，参数列表中的index是第i个字母的意思；
每个itemHead高度是itemHeadHeight，共有groupIndex个；
每个Item高度是itemHeight，共有itemIndex个；

先通过sublist获取第1到第i个字母中间所有的Postal[]，然后对其长度进行求和，就可以得到itemIndex，而groupIndex就是字母顺序；
![示意图](/img/posts/studys-zero-mall/03-08-image-args-postallistwidget.png)
### 定位信息
使用*location*获取经纬度信息
使用*flutter_z_location*转换经纬度为省县市
``` dart
try {
  // 获取GPS定位经纬度
  Location location = new Location();
  bool _serviceEnabled;
  PermissionStatus _permissionGranted;
  LocationData _locationData;

  _serviceEnabled = await location.serviceEnabled();
  if (!_serviceEnabled) {
    _serviceEnabled =
        await location.requestService();
    if (!_serviceEnabled) {
      print("服务没启动");
      return;
    }
  }

  _permissionGranted =
      await location.hasPermission();
  if (_permissionGranted ==
      PermissionStatus.denied) {
    _permissionGranted =
        await location.requestPermission();
    if (_permissionGranted !=
        PermissionStatus.granted) {
      print("请求权限没成功");
      return;
    }
  }
  _locationData = await location.getLocation();

  print(
      "loc:${_locationData.latitude},${_locationData.longitude}");
  // 经纬度反向地理编码获取地址信息(省、市、区)
  final loc =
      await FlutterZLocation.geocodeCoordinate(
          _locationData.latitude!,
          _locationData.longitude!,
          pathHead: 'assets/');
  setState(() {
    _postalCode = loc.districtId;
    postalList[0] = loc.province;
    postalList[1] = loc.city;
    postalList[2] = loc.district;
    _postalRegion.text = _postalRegionDesc;
  });
} catch (e) {
  print("获取地理信息失败了:$e");
}
```
### 通讯录
``` dart
Expanded(
    child: IconButton(
  icon: const Icon(Icons.contact_phone),
  color: Colors.blue,
  onPressed: () async {
    try {
      final PhoneContact contact =
          await FlutterContactPicker.pickPhoneContact();
      setState(() {
        _phoneNumber.text = contact.phoneNumber!.number!;
        _name.text = contact.fullName!;
      });
    } catch (e) {
      print("选择失败");
    }
  },
))
```
使用通讯录获取手机号和名字

### 地址粘贴板
新增页面增加一个地址粘贴板的功能
1. 初步提取手机号、姓名
   1. 使用多种分隔符分割字符串
   2. 遍历数组，寻找手机号
   3. 遍历数组，寻找姓名（姓氏开头，2-4位名字）
2. 提取所在区域信息
   1. 遍历*Tables*，判断文本中是否含有邮递区域相关的信息
   2. 找到最符合的地址，并设置 所在区域 信息
3. 最终提取信息
   1. 将1. 2.获取到的信息全部替换成空格
   2. 使用多种分隔符分割字符串
   3. 当1. 没有找到手机号时，遍历数组，寻找手机号
   4. 当1. 没有找到手机号时，遍历数组，寻找姓名（姓氏开头，2-4位名字）
   5. 遍历数组，将最长的一项设置为详细地址🤣

{% gallery %}
![改造后](/img/posts/studys-zero-mall/03-06-image-lookup-user-addressinfo.png)
{% endgallery %}

## 可能的其他改进

1. 所在区域选择的时候，可仿造京东提供热门城市的选项
2. 一些细节并未处理
3. 可接入高德地图等API，在地图上选择位置


