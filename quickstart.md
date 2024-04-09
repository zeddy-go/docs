# 快速开始
## http服务
   克隆代码到本地
   ```bash
   git clone -b step1 --single-branch git@github.com:zeddy-go/quickstart.git
   ```
   我们可以看到如下目录结构
   ```
   .
   ├── go.mod
   ├── go.sum
   ├── main.go                         //入口文件
   └── module                          //模块
       └── user                        //用户模块
           ├── iface                   //接口层
           │   └── http          //http接口
           │       └── user.go   //user handler
           └── module.go               //模块入口
   ```
   在这个项目中，我们引用了 ginx 模块和 user 模块。
   ginx 模块为框架自带模块，用于处理 http 请求，底层使用 gin 。
   > 您完全可以使用任何您喜欢的http库编写自己的http处理模块。或者自己造轮子:100:。
   
   user 模块则是我们自己的业务模块。
   user 模块实现了一个 handler 方法，当接收到请求时，该方法会输出 hello xxxx! 的JSON响应(module/user/iface/http/user.go)。
   接着，它向 ginx 模块注册了一个路由，以接收对应的请求(module/user/module.go:30)。
   
   在这里你会注意到 container.Invoke 方法。没错，框架使用了`依赖注入`。
   我们正尝试在开发效率和执行效率之间找到平衡。
   而对象的实例化方法则是在上面的Init方法中进行绑定(module/user/module.go:20)。

   > Note: 请在Init方法中绑定实例化方法，在Boot方法中或其他地方使用他们。

## 数据库相关操作
克隆代码到本地
   ```bash
   git clone -b step2 --single-branch git@github.com:zeddy-go/quickstart.git
   ```
