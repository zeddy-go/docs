# 快速开始
## http服务
   克隆代码到本地
   ```bash
   git clone -b step1 --single-branch git@github.com:zeddy-go/quickstart.git
   ```
   我们可以看到如下目录结构
   ```
   .
   ├── conf                       //配置
   │   ├── config.go
   │   └── config.yaml
   ├── go.mod
   ├── go.sum
   ├── main.go                     //入口文件
   └── module                      //模块
       └── user                    //用户模块
           ├── iface               //接口层
           │     └── http          //http接口
           │         └── user.go   //user handler
           └── module.go           //模块入口
   ```
   我们使用embed的方式来加载配置，然后通过环境变量来对配置做改动。
   这种方式在容器化部署的场景下非常方便。
   当然，您仍然可以按照您喜欢的方式来做。

   我们引用了 ginx 模块和 user 模块。
   ginx 模块为框架内置模块，用于处理 http 请求，底层使用 gin 。
   > 您完全可以使用任何您喜欢的http库编写自己的http处理模块。或者自己造轮子:100:。
   
   user 模块则是我们自己的业务模块。
   user 模块实现了一个 handler 方法，当接收到请求时，该方法会输出 hello xxxx! 的JSON响应(module/user/iface/http/user.go)。
   接着，它向 ginx 模块注册了一个路由，以接收对应的请求(module/user/module.go:30)。
   
   在这里您会注意到 container.Invoke 方法。没错，框架使用了`依赖注入`。
   (我们正尝试在开发效率和执行效率之间找到平衡。)
   而对象的实例化方法则是在上面的Init方法中进行绑定(module/user/module.go:21)。

   > Note: 请在Init方法中绑定实例化方法，在Boot方法中或其他地方使用他们。

   启动服务 `go run .` ，然后访问 http://localhost:8080/hello?username=zed 。
## 更完整的例子
克隆代码到本地
   ```bash
   git clone -b step2 --single-branch git@github.com:zeddy-go/quickstart.git
   ```
在这个例子中，目录结构如下:
   ```
   .
   ├── config                                //配置
   │   ├── config.go
   │   └── config.yaml
   ├── docker-compose.yaml                   //这个例子使用docker来启动MySQL数据库
   ├── go.mod
   ├── go.sum
   ├── main.go                               //入口文件
   └── module
       └── user
           ├── domain                        //领域层
           │         ├── contract.go         //一些接口
           │         ├── svc                 //服务
           │         │     └── user.go
           │         └── user.go             //领域实体
           ├── iface                         //接口层
           │         └── http
           │               ├── payload.go
           │               └── user.go       //用户 handler
           ├── infra
           │         ├── migration           //迁移
           │         │     ├── 20240408063653_create_users_table.down.sql
           │         │     ├── 20240408063653_create_users_table.up.sql
           │         │     └── migration.go
           │         ├── model               //数据模型
           │         │     └── user.go
           │         ├── repo                //仓库
           │         │     └── user.go
           │         └── seed                //数据填充
           │               └── user.go
           └── module.go                     //模块入口
   ```
### 数据操作相关
在这个例子中数据模型与领域实体相似，但在实际项目中，一个领域实体也可能由多个数据模型的数据组成。
业务逻辑只使用领域实体，所以在通过仓库对数据做操作时，仓库会帮我们进行双向转换。
> 仓库内置了简单的转换逻辑，但无法满足复杂情况。这时就需要自定义转换逻辑。

### 数据迁移
框架内置的 migration 模块使用 github.com/golang-migrate/migrate 包实现迁移。
这同样需要在模块入口中注册(module/user/module.go:41)。程序启动后，数据表会自动创建到数据库。

### 数据填充
框架内置了数据填充，原理很简单，就是利用容器执行函数。
这同样需要在模块入口中注册(module/user/module.go:42)。
程序启动后，数据会自动填充到数据库。

### user handler
在 user handler 中，我们添加了两个方法，一个是创建用户的方法，一个是获取用户信息的方法。
与第一个例子一样，我们在 module.go 中也新注册了两个路由。

您会发现这两个新方法与第一个例子中的方法的返回值数量不一样。
这正是框架的另一个特性，handler函数的可变参数和返回值。
框架允许您为每个handler函数设计自己的参数和返回值。
对于参数，框架会遍历参数列表，查找容器中的对象(依赖注入)或者解析请求的参数。
对于返回值，框架会遍历返回值列表，来决定如何返回响应。

## 添加grpc服务
克隆代码到本地
```bash
git clone -b step3 --single-branch git@github.com:zeddy-go/quickstart.git
```
我们在原有的项目的基础上添加grpc服务，这样我们的服务不仅能服务http也能同时服务grpc。
当然，如果您的项目只需要基于grpc，您完全可以不用理会http相关的代码。
项目目录结构如下:
```
.
├── conf
│    ├── config.go
│    └── config.yaml
├── docker-compose.yaml
├── go.mod
├── go.sum
├── main.go
├── module
│       └── user
│             ├── domain
│             │       ├── contract.go
│             │       ├── svc
│             │       │     └── user.go
│             │       └── user.go
│             ├── iface
│             │       ├── grpc
│             │       │     └── user.go            //grpc接口实现
│             │       └── http
│             │           ├── payload.go
│             │           └── user.go
│             ├── infra
│             │       ├── migration
│             │       │     ├── 20240408063653_create_users_table.down.sql
│             │       │     ├── 20240408063653_create_users_table.up.sql
│             │       │     └── migration.go
│             │       ├── model
│             │       │     └── user.go
│             │       ├── repo
│             │       │     └── user.go
│             │       └── seed
│             │             └── user.go
│             └── module.go
├── pb                                             //pb文件
│    ├── user_grpc.pb.go
│    └── user.pb.go
├── readme.md
└── user.proto                                     //proto文件
```
在 main.go 文件中，我们添加内置的 grpc 模块(main.go:21)。
而注册我们自己的服务，只需在 module/user/module.go 文件中从容器中拿到已实例化好的 *grpc.Server 即可(module/user/module.go:44,module/user/module.go:65)。

> 这里不得不再次提到依赖注入。您完全可以将您的常用代码以`框架模块`的形式固化起来，再一股脑的塞到容器里，随用随取。
容器中的每个对象默认为单例模式，得益于常驻内存，已实例化的对象将不再重复实例化，这对执行效率来说是一件好事。
同时我们也期待您将自己的`框架模块`开源，共同构建一个便捷的，包罗万象的生态环境。

另外，利用已有代码添加grpc还能非常方便的展示代码重用的特性。这个特性不是框架的功劳，而是ddd的。
您可以看到，在grpc接口实现文件里面，我们重用了 userService 。

> 顺带一提，github.com/zeddy-go/zeddy/errx 包默默的收集了错误栈，您可以使用这样的代码来打印它 fmt.Printf("%#v", err) 。
> 并且，这些信息在经过grpc传输后也不会丢失。
