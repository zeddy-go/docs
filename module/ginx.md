# ginx

ginx模块用于使用给定的handler处理相应的请求，底层使用gin。
该模块封装了gin的路由相关操作，实现了不限制handler方法的入参和出参列表的特性，最大的好处是降低了心智负担。
ginx模块在实例化时提供两个option方法：
* ginx.WithCustomEngine 用于使用自定义的gin实例
* ginx.WithPrefix 用于设置读取配置的层级，默认是根层级

ginx模块在初始化时会将自己以 ginx.Router 的接口类型注册到容器中，
其他需要使用http能力的模块只需在自己的Boot方法中拿到 ginx.Router 对象，添加路由即可。

## handler 入参
ginx 在将请求转到对应的handler时，会通过反射遍历入参。
如果参数类型在容器中能找到则使用容器中的对象传参，
如果参数类型在容器中无法找到，则尝试解析请求获得。
```go
//-----------userHandler-----------------
//解析请求
type HelloReq struct {  //与gin的定义完全一样
    Username string `form:"username" binding:"required"`
}
type HelloResp struct {
    Text string `json:"text"`
}

func (uh *UserHandler) Hello(req *HelloReq) {}

//从容器中获取对象
func (uh *UserHandler) Hello(c *viper.Viper) {}

//也可以全都要,顺序无关
func (uh *UserHandler) Hello(req *HelloReq, c *viper.Viper) {}

//----------------module.Boot--------------------
err = container.Invoke(func(r ginx.Router, userHandler *http.UserHandler) {
    r.GET("/hello", userHandler.Hello)
})
if err != nil {
    return
}
```

另外，有些ginx定了一些特定的入参类型，来达到简化代码的效果：
```go
// 直接获取gin的Context对象
func (uh *UserHandler) Hello(ctx *gin.Context) {}

// 获取分页, 解析url query-string 中的 page 和 size两个字段。
func (uh *UserHandler) Hello(page *ginx.Page) {}

// 获取筛选条件, 解析url query-string 中所有形如 filters[xxx] 的字段
// ginx.Filters类型可将这些字段转为 gormx 模块可理解的查询条件
func (uh *UserHandler) Hello(filters *ginx.Filters) {}

// 获取排序条件, 解析url query-string 中所有形如 sorts[xxx]的字段
// ginx.Sorts类型可将这些字段转为 gromx 模块可理解的排序条件
func (uh *UserHandler) Hello(sorts *ginx.Sorts) {}
```
与上面的例子相同，这里的例子也可以一起作为同一个handler的参数。

## handler 出参
虽然出参仍然是可变的，但是为了ginx能正常处理出参并合适的返回响应，出参也是需要遵循一定的规则的。

ginx 默认的响应结构体遵循 Restful 风格。

首先，可以没有出参，这种情况 ginx 将认为函数一定会执行成功，并且没有响应体。返回的状态码是204。
```
HttpStatusCode: 204
body: (无)
```

在有出参的情况下，最后一个出参必须永远是 error 类型。ginx 将会判断 error 是否为nil，来判断响应的状态吗。
当error不为nil时，默认状态码为500。
```
HttpStatusCode: 500
body: {"message": err.Error(), "data": null}
```
如果您使用 errx 包返回的错误，您将能自定义状态码，和详情：
```go
errx.New("数据正在被使用，无法删除", errx.WithDetailMap(map[string]any{"detail": "this is detail"}), errx.WithCode(http.StatusConflict))  //这将会设置状态码为409
```
```
HttpStatusCode: 409
body: {"message": "数据正在被使用，无法删除", "data": {"detail": "this is detail"}}
```

也就是说，当出参为1个时，这个出参一定为 error 类型。

当出参为2个时，最后一个一定是error类型，而第一个则是您想要返回的数据。当error不为nil时，与上面的例子相同。
当error为nil时效果如下：
```
HttpStatusCode: 200
body: {"message": "", "data": <您的响应数据>}
```

当出参为3个时，error的规定与上面一致，第1个出参为元信息，第2个出参为您想返回的数据。
元信息为实现了ginx.IMeta接口的结构体或一个整数(int)。整数表示分页的总记录数：
```
func (uh *UserHandler) Hello(page *ginx.Page) (total int, list []*Xxx, err error) {}

将会返回：
HttpStatusCode: 200
body: {"message": "", "data": <您的响应数据>, "meta": {"total": <number>}}
```
而 ginx.IMeta 接口的结构体将会被转换成json字符串，其中如果有 code 字段，该字段将会作为状态码返回：
```
func (uh *UserHandler) Hello(page *ginx.Page) (meta ginx.IMeta, list []*any, err error) {
	return &ginx.Meta{CurrentPage: 1, Code: http.StatusCreated}, []*any{}, nil
}

将会返回：
HttpStatusCode: 201
body: {"message": "", "data": <您的响应数据>, "meta": {"currentPage": 1}}
```

出参最多只能有3个。

那么，如何定义文件下载呢？代码如下：
```go
func (uh *UserHandler) Hello(req *HelloReq) (file ginx.IFile, err error) {
    f, err := os.Open("/path/to/file.mp3")
    if err != nil {
        return
    }
    defer f.Close()
    content, err := io.ReadAll(f)
    if err != nil {
        return
    }
    file = ginx.NewFile(content)
    return
}
```
