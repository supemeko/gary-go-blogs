---
title: Flutter电商项目学习笔记 02-完善后管错误处理
date: 2024-04-26 16:21:11
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
### 发生了什么
后台管理系统的前端页面，没有处理服务端响应中的错误信息。
后端响应{% label "200 OK" blue %}时，前端没有处理http应答中包含的错误信息。

### 复现步骤
1. 停止redis服务
2. 登录后台管理系统
3. 打开 {% label '系统管理 > 用户列表' blue %} 页面
4. 用户列表页面无数据且无错误提示

经过观察，几乎所有页面，都不会对http应答中以下信息作出反应
`{code: 1001, message:'用户：admin,获取redis连接异常'}`

### 文章内容
1. 后端应用分析
2. 前端应用分析
3. 多种解决方案及最终的选择

## 后端应用分析
### 后端错误分析
#### 日志信息
```text
{"@timestamp":"2024-04-26T16:49:59.518+08:00","caller":"middleware/addloglmiddleware.go:47","content":"Request: GET /api/sys/user/info ","level":"info","span":"bdacf28d95d43c4f","trace":"5c9f929db15eeb10f7d1b0edb13e0a6f"}
{"@timestamp":"2024-04-26T16:49:59.599+08:00","caller":"user/userinfologic.go:92","content":"设置用户：admin,权限到redis异常: dial tcp 127.0.0.1:6379: connect: connection refused","level":"error","span":"bdacf28d95d43c4f","trace":"5c9f929db15eeb10f7d1b0edb13e0a6f"}
{"@timestamp":"2024-04-26T16:49:59.605+08:00","caller":"handler/loghandler.go:147","content":"[HTTP] 200 - GET /api/sys/user/info - 192.168.10.20:36636 - Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36","duration":"86.9ms","level":"info","span":"bdacf28d95d43c4f","trace":"5c9f929db15eeb10f7d1b0edb13e0a6f"}
{"@timestamp":"2024-04-26T16:49:59.867+08:00","caller":"middleware/addloglmiddleware.go:47","content":"Request: POST /api/sys/user/list {\"current\":1,\"pageSize\":10}","level":"info","span":"1e011e6d613d88e1","trace":"c2807e5276595a13ee9af9f5fb62dd61"}
{"@timestamp":"2024-04-26T16:49:59.927+08:00","caller":"middleware/checkurlmiddleware.go:43","content":"用户：admin,获取redis连接异常","level":"error","span":"1e011e6d613d88e1","trace":"c2807e5276595a13ee9af9f5fb62dd61"}
{"@timestamp":"2024-04-26T16:49:59.935+08:00","caller":"handler/loghandler.go:147","content":"[HTTP] 200 - POST /api/sys/user/list - 192.168.10.20:36650 - Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36","duration":"67.6ms","level":"info","span":"1e011e6d613d88e1","trace":"c2807e5276595a13ee9af9f5fb62dd61"}
```
#### 日志分析
日志报告了两则消息,下面对两则消息进行分析：
- 日志ID`5c9f929db15eeb10f7d1b0edb13e0a6f`
  - `/api/sys/user/info` 该接口用于登录后请求用户信息
  - 日志报告`设置用户：admin,权限到redis异常`
  - 应答 200 GET
- 日志ID`c2807e5276595a13ee9af9f5fb62dd61`
  - `/api/sys/user/list` 该接口用于响应{% label '系统管理 > 用户列表' blue %}的用户信息请求
  - 日志报告`用户：admin,获取redis连接异常`
  - 应答 200 POST

#### 错误分析
经排查，用户登录接口中并未获取{% label 可访问的权限列表 blue %}，前端会另外通过`/api/sys/user/info`接口主动查询权限信息。`/api/sys/user/info`从数据库获取当前用户的权限信息后，会将权限信息保存到redis中，并将其返回给前端。当redis并未启动时，权限信息保存失败，但是接口仍返回从数据库中获取到的权限信息。

后续用户访问{% label '系统管理 > 用户列表' blue %}界面，访问`/api/sys/user/list`接口，接口从redis中查询信息失败，接口仍返回前端200 POST，请求成功。从浏览器控制台发现接口返回内容为`{"code":1001,"message":"用户：admin,获取redis连接异常"}`，此时能够确定问题来自前端未能正确处理`Responce.code='1001'`。

### 后端分析过程学习成果
#### go-zero 路由设置
``` go api/admin/internal/handler/routes.go
server.AddRoutes(
    rest.WithMiddlewares(
        []rest.Middleware{serverCtx.CheckUrl},
        []rest.Route{
            // 其他API路由...
            {
                Method:  http.MethodGet,
                Path:    "/info",
                Handler: sysuser.UserInfoHandler(serverCtx),
            },
            {
                Method:  http.MethodPost,
                Path:    "/list",
                Handler: sysuser.UserListHandler(serverCtx),
            },
        }...,
    ),
    rest.WithJwt(serverCtx.Config.Auth.AccessSecret),
    rest.WithPrefix("/api/sys/user"),
)
```
可以看到当连接进入`/api/sys/user/list`和`/api/sys/user/info`时会进入【权限检查中间件】(`serverCtx.CheckUrl`)流程，注意接口会经过Jwt校验

#### 权限检查中间件
{% tabs 权限检查中间件, 1 %}
<!-- tab 关键代码 -->
{% codeblock api/admin/internal/middleware/checkurlmiddleware.go lang:go highlight:true %}
func (m *CheckUrlMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {

    // 其他代码...

    if r.RequestURI == "/api/sys/user/info" || r.RequestURI == "/api/sys/user/queryAllRelations" || r.RequestURI == "/api/sys/role/queryMenuByRoleId" {
      next(w, r)
      return
    }

    //获取用户能访问的url
    urls, err := m.Redis.Get(userId)
    if err != nil {
      logx.WithContext(r.Context()).Errorf("用户：%s,获取redis连接异常", userName)
      httpx.Error(w, errorx.NewDefaultError(fmt.Sprintf("用户：%s,获取redis连接异常", userName)))
      return
    }

    // 其他代码...
  }
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab 完整代码 -->
{% codeblock api/admin/internal/middleware/checkurlmiddleware.go lang:go highlight:true mark:18-24 %}
func (m *CheckUrlMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {

    //判断请求header中是否携带了x-user-id
    userId := r.Context().Value("userId").(json.Number).String()
    userName := r.Context().Value("userName").(string)
    if userId == "" || userName == "" {
      logx.WithContext(r.Context()).Errorf("缺少必要参数x-user-id")
      httpx.Error(w, errorx.NewDefaultError("缺少必要参数x-user-id"))
      return
    }

    if r.RequestURI == "/api/sys/user/info" || r.RequestURI == "/api/sys/user/queryAllRelations" || r.RequestURI == "/api/sys/role/queryMenuByRoleId" {
      next(w, r)
      return
    }

    //获取用户能访问的url
    urls, err := m.Redis.Get(userId)
    if err != nil {
      logx.WithContext(r.Context()).Errorf("用户：%s,获取redis连接异常", userName)
      httpx.Error(w, errorx.NewDefaultError(fmt.Sprintf("用户：%s,获取redis连接异常", userName)))
      return
    }

    if len(strings.TrimSpace(urls)) == 0 {
      logx.WithContext(r.Context()).Errorf("用户: %s,还没有登录", userName)
      httpx.Error(w, errorx.NewDefaultError(fmt.Sprintf("用户: %s,还没有登录,请先登录", userName)))
      return
    }

    backUrls := strings.Split(urls, ",")

    b := false
    for _, url := range backUrls {
      if url == r.RequestURI {
        b = true
        break
      }
    }

    if !b {
      logx.WithContext(r.Context()).Errorf("用户: %s,没有访问: %s路径的权限", userName, r.RequestURI)
      httpx.Error(w, errorx.NewDefaultError(fmt.Sprintf("用户: %s,没有访问: %s,路径的的权限,请联系管理员", userName, r.RequestURI)))
      return
    }

    next(w, r)
  }
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}
注意到go-zero通过**httpx.Error**设置http应答，该函数签名为`func Error(w http.ResponseWriter, err error, ...)`
函数参数列表中的**err**会通过**handler**(`func(error) (int, any)`)转化为http应答码和应答信息，而参数列表中的**w**将转化后的应答码和应答信息设置为当前请求的http响应。
上面提到的**handler**，由**httpx**包的另一个函数**httpx.SetErrorHandler**指定。

#### 设置httpx ErrorHandler
对于Admin-Api（负责本次请求的处理的微服务，{% btn 'https://feihua.github.io/mall/',微服务结构拓扑图,far fa-hand-point-right, outline blue  %}）来说，**handler**通过以下代码设置：
``` go 
httpx.SetErrorHandler(func(err error) (int, interface{}) {
    switch e := err.(type) {
    case *errorx.CodeError:
        return http.StatusOK, e.Data()
    default:
        return http.StatusInternalServerError, nil
    }
})
```
**http.StatusOK** 为 {% label "200 OK" blue %}
**http.StatusInternalServerError** 为 {% label "500 Internal Server Error" red %}
**e.Data()** 将用于应答信息的设置

该handler的功能如下：
1. 如果错误的类型为**errorx.CodeError**，则返回 {% label "200 OK" blue %}，并将e.Data()作为应答信息；
2. 否则返回{% label "500 Internal Server Error" red %}，并且不设置应答信息；

#### 本次请求中的http响应

本次请求通过httpx.Error设置http响应
`
httpx.Error(w, errorx.NewDefaultError(fmt.Sprintf("用户：%s,获取redis连接异常", userName)))
`

**errorx.NewDefaultError**将生成一个**error**，该**error**可以用json表示为
``` json
{
    code: "1001",
    message: "用户：%s,获取redis连接异常",
}
```
1001是**NewDefaultError**指定的默认应答码。
其错误类型为`errorx.CodeError`,将被映射成{% label "200 OK" blue %}
🔔注意：200是http响应码，1001是服务器内部定义的应答码，不要搞混了。

#### httpx的接口设计
httpx: https://pkg.go.dev/github.com/zeromicro/go-zero@v1.6.3/rest/httpx
``` go httpx https://pkg.go.dev/github.com/zeromicro/go-zero@v1.6.3/rest/httpx
func Error(w http.ResponseWriter, err error, ...)
func ErrorCtx(ctx context.Context, w http.ResponseWriter, err error, ...)
func GetFormValues(r *http.Request) (map[string]any, error)
func GetRemoteAddr(r *http.Request) string
func Ok(w http.ResponseWriter)
func OkJson(w http.ResponseWriter, v any)
func OkJsonCtx(ctx context.Context, w http.ResponseWriter, v any)
func Parse(r *http.Request, v any) error
func ParseForm(r *http.Request, v any) error
func ParseHeader(headerValue string) map[string]string
func ParseHeaders(r *http.Request, v any) error
func ParseJsonBody(r *http.Request, v any) error
func ParsePath(r *http.Request, v any) error
func SetErrorHandler(handler func(error) (int, any))
func SetErrorHandlerCtx(handlerCtx func(context.Context, error) (int, any))
func SetOkHandler(handler func(context.Context, any) any)
func SetValidator(val Validator)
func WriteJson(w http.ResponseWriter, code int, v any)
func WriteJsonCtx(ctx context.Context, w http.ResponseWriter, code int, v any)
type Router
type Validator
```
- 这个包的接口设计清晰，简单合理。👍
- `func WriteJson(w http.ResponseWriter, code int, v any)` 用于将上文中的`e.Data()`解释为Json串
- 上文中提到的`Error`,`SetErrorHandler`,`WriteJson`，均有携带http请求上下文的版本，分别是`ErrorCtx`, `SetErrorHandlerCtx`, `WriteJsonCtx`

### 前端应用分析

#### **plugin-request**的使用
前端向后端请求服务端数据，这些请求有时需要进行统一的处理，比如添加**Http Header**信息，在项目中通过在`src/app.tsx`中设置`export const request: RequestConfig`来配置
``` tsx src/app.tsx
export const request: RequestConfig = {
  //...
};
```

**RequestConfig**是**plugin-request**的接口，可以用来配置请求拦截器，响应拦截器，自定义中间件。`plugin-request`默认提供了统一错误处理的中间件服务。
``` ts @@/plugin-request/request.ts
export interface RequestConfig extends RequestOptionsInit {
  errorConfig?: {
    errorPage?: string;
    adaptor?: (resData: any, ctx: Context) => ErrorInfoStructure;
  };
  middlewares?: OnionMiddleware[];
  requestInterceptors?: RequestInterceptor[];
  responseInterceptors?: ResponseInterceptor[];
}
```

#### 用户列表页面请求后端
##### 代码目录
以下是前端中{% label '系统管理 > 用户列表' blue %}的代码目录，**service.ts**负责定义向后端接口的请求，**index.tsx**负责描述页面。
```
src/pages/system/user/
├── components
│   ├── CreateUserForm.tsx
│   └── UpdateUserForm.tsx
├── data.d.ts
├── index.tsx
└── service.ts
```

##### service.ts & plugin-request
在**service.ts**中，查询用户列表通过一下方式定义，其中**request**方法会经由**plugin-request**处理。
``` ts
//查询用户列表
export async function queryUserList(params: UserListParams) {
  if (params.status != null) {
    params.status = Number(params.status);
  }
  return request('/api/sys/user/list', {
    method: 'POST',
    data: {
      ...params,
    },
  });
}
```

##### index.tsx & ProTable
在**index.tsx**中，使用**ProTable**组件展示画面。
**ProTable**是**ProComponents**提供的组件，在需要请求用户列表分页数据时会调用其中**request**定义的方法**queryUserList**。
``` tsx
<ProTable<UserListItem>
  //...
  request={queryUserList}
  //...
/>
```

#### 错误分析

##### 请求后端接口过程
页面组件**ProTable**通过调用**queryUserList**请求后端数据，**queryUserList**将调用**plugin-request**的**request**方法进行处理，其中会经过**plugin-request**设置的拦截器，其中包括一个错误处理。

``` js
{
  data: T[],
  success: true,
  total: number,
}
```
以上是页面部件**ProTable**需要接收的数据类型，其中：
- success设置是否成功的标志位 “请返回 true，不然 table 会停止解析数据，即使有数据”
- data用于更新页面显示的数据
- total用于页面部件设置分页

##### 成功返回页面数据
``` go
type ListUserResp struct {
	Code     string          `json:"code"`
	Message  string          `json:"message"`
	Current  int64           `json:"current,default=1"`
	Data     []*ListUserData `json:"data"`
	PageSize int64           `json:"pageSize,default=20"`
	Success  bool            `json:"success"`
	Total    int64           `json:"total"`
}
```
后端请求成功时页面组件收到的数据类型，代码是go，通过json传输，前端将json转换成对应js对象，**plugin-request**不会更改对象的内容，**ProTable**只处理其中Data、Success、Total字段。

##### 后端处理异常
``` go
type CodeErrorResponse struct {
	Code    int    `json:"code"`
	Message string `json:"message"`
}
```
后端处理异常时，后端返回前端{% label "200 OK" blue %}, **plugin-request**同样不会更改对象的内容。由于页面组件收到的消息缺少`success: true`，因此页面不会产生变化。但是也没有默认进行错误处理。

##### http响应异常
``` js 项目中定义的统一错误处理
const errorHandler = (error: any) => {
  const { response, name, info } = error;
  if (response && response.status) {
    const errorText = codeMessage[response.status] || response.statusText;
    const { status, url } = response;

    notification.error({
      message: `请求错误 ${status}: ${url}`,
      description: errorText,
    });
    throw error;
  }
  // ...其他代码
};
```
当“502 网关超时”或者“404 找不到资源”时，**plugin-request**将抛出异常，并在用户定义的统一错误处理阶段进行处理。
上面这段程序中,**error.response.status**是http响应码,**codeMessage**负责将http响应码转化成对应文字描述。
**notification.error**负责将弹出浮窗向用户报告错误：
{% gallery %}
![错误复现](/img/posts/studys-zero-mall/02-02-image-2.png)
{% endgallery %}

## 多种解决方案及最终的选择

### 可能的几种方案，以及优缺点
1. 修改后端响应，使其在错误是返回非200，引起前端错误，引导程序弹出浮窗向用户报告错误
 - 涉及http响应码，增加复杂度。
 - 在不同的层次封装业务异常，难以理解。
 - 实现简单，只需要更改对应http应答码映射，将业务异常映射为500应答码。

2. 统一前后端传输格式，通过专门的数据格式，使得前端能够进行统一错误业务异常。
 - 分层混乱，框架需要定义错误处理格式，用户自己又定义错误处理格式，页面又有自己的响应逻辑，这样错误处理就经过太多层错误处理了
 - 不够正确，业务异常的处理实际上是由页面定义的，而非“统一错误处理”。
 - 不利于前后端分离，处理特殊接口时又要避开某些统一处理机制

3. 由页面负责处理业务响应码
 - 每个页面都需要编码改造成本大
 - 页面组件没有提供方便的处理接口

### 最终选择：
 - 后端响应已包含code, message两个字段，利用这两个字段对请求进行统一错误处理。
 - 个别请求将绕过统一错误处理机制，进行个性化处理，逐步改造所有页面
 - http应答码非200 OK时，逻辑处理不变

### 实现方法

#### 配置**plugin-request**统一错误处理
##### 设置**RequestConfig**
``` ts
export const request: RequestConfig = {
  errorConfig: {
    adaptor: (res, ctx) => {
      // success为false时
      // 函数返回后，umi会抛出一个RequestError
      // 幸运的话，会被RequestConfig.errorHandler接收
      // suceess为true时，什么都不会发生
      return {
        success: res.code == '000000',
        data: res,
        errorCode: res.code,
        errorMessage: res.message,
        showType: ErrorShowType.NOTIFICATION,
        url: ctx?.req?.url,
        method: ctx?.req?.options?.method,
      };
    },
  },
  // ...其他代码
}
```
设置**RequestConfig.errorConfig.adaptor**，将从**http响应**中提取错误信息

##### **plugin-request**将抛出**BizError**异常
``` ts
  // 中间件统一错误处理
  // 后端返回格式 { success: boolean, data: any }
  // 按照项目具体情况修改该部分逻辑
  requestMethodInstance.use(async (ctx, next) => {
    await next();
    const { req, res } = ctx;
    // @ts-ignore
    if (req.options?.skipErrorHandler) {
      return;
    }
    const { options } = req;
    const { getResponse } = options;
    const resData = getResponse ? res.data : res;
    const errorInfo = errorAdaptor(resData, ctx);
    if (errorInfo.success === false) {
      // 抛出错误到 errorHandler 中处理
      const error: RequestError = new Error(errorInfo.errorMessage);
      error.name = 'BizError';
      error.data = resData;
      error.info = errorInfo;
      error.response = res;
      throw error;
    }
  });
```
如果错误信息中**success**为**false**，将抛出**BizError**类型的异常，该异常将被统一错误处理捕获。

##### 设置**plugin-request**统一错误处理代码
``` tsx 针对BizError类型的异常进行错误处理
const errorHandler = (error: any) => {
  //RequestConfig.errorConfig.adaptor定义的业务异常
  if (name == 'BizError' && info && info.errorMessage && info.errorCode) {
    const url = info?.url;
    const method = info?.method;
    const req_info = (method ? method : '请求接口') + ' ' + url;
    const req_info_display = url ? req_info : null;
    notification.error({
      message: '后端处理异常',
      description: (
        <>
          <p>
            {req_info_display && (
              <>
                {req_info_display}
                <br />
              </>
            )}
            后端应答码 [{info.errorCode}]<br />
            后端应答信息: 【{info.errorMessage}】
          </p>
        </>
      ),
    });
    console.error(
      `后端处理异常 Request(${req_info_display}) ErrorCode(${info.errorCode}) ErrorMsg(${info.errorMessage})`,
    );
    return;
  }
  // ...其他代码
}
```

#### 设置统一错误处理效果
页面中弹出浮窗提示用户发生异常
{% gallery %}
![浮窗提示](/img/posts/studys-zero-mall/02-02-image-2.png)
{% endgallery %}

控制台中打印人性化的错误信息
{% gallery %}
![控制台错误输出](/img/posts/studys-zero-mall/02-04-image-4.png)
{% endgallery %}

#### 实现页面级错误处理
##### 绕过**plugin-request**统一错误处理
``` ts
//查询用户列表
export async function queryUserList(params: UserListParams) {
  if (params.status != null) {
    params.status = Number(params.status);
  }

  return request('/api/sys/user/list', {
    method: 'POST',
    data: {
      ...params,
    },
    skipErrorHandler: true, // 跳过统一错误处理
  });
}
```
在**service.ts**中,增加请求选项**skipErrorHandler: true**

##### 设置**ProTable**请求分页数据的方法
``` ts
request={async (params) => {
  const res = await queryUserList(params);
  if (!res.success) {
    error({
      title: '后端处理异常',
      content: (
        <p>
          后端应答码 [{res.code}]<br />
          后端应答信息 【{res.message}】
        </p>
      ),
    });
  }
  return res;
}}
```
修改**ProTable**的**request**字段，当http响应中的**success**不存在或不为true时，弹窗警告用户。

#### 页面级错误处理效果
{% gallery %}
![弹窗警告](/img/posts/studys-zero-mall/02-03-image-3.png)
{% endgallery %}

## 写在后面
下一章节将完善移动端地址功能
