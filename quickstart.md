# 快速开始
## 初始化一个项目
1. 初始化一个项目
    ```bash
    mkdir quickstart && cd quickstart && go mod init quickstart
    ```
2. 引入框架库
    ```bash
    go get -u github.com/zeddy-go/zeddy
    ```
3. 将下面的代码复制到main.go中
    ```go
    package main
    
    import (
    	"github.com/zeddy-go/zeddy/app"
    	"log/slog"
    )
    
    func main() {
    	err := app.StartAndWait()
    	if err != nil {
    		slog.Warn("An error occurred", "error", err)
    	}
    }
    ```

4. 跑起来
    ```bash
    go run .
    ```
    这时,程序会输出类似如下的信息,并且停止:
    ```shell
    2023/12/29 16:42:03 INFO nothing started, shutdown.
    ```
    第一个zeddy程序已经成功运行了~

    > 如你所见，程序仅仅输出了一条日志便退出了。这是因为我们还没有往程序中添加 `模块`。zeddy 的目标是提供一个能够快速上手，
    并且完全由你掌控的开发体验。所以一个初始的 zeddy 程序并不会附加任何功能。

## 编写模块
### http api
1. 新建文件 module/user/module.go
2. 复制以下代码:
    ```go
    package user
    
    import (
        "github.com/zeddy-go/zeddy/module"
    )
    func NewModule() *Module {
        return &Module{} //模块名称目前不是必须的，所以可以不用自己实例化BaseModule
    }
    
    type Module struct {
        module.BaseModule
    }
    ```

3. 创建api
    * 新建文件 module/user/iface/http/user.go
    * 复制以下代码
        ```go
        package http

        func NewUserHandler() *UserHandler {
            return &UserHandler{}
        }
    
        type UserHandler struct{}
    
        func (uh *UserHandler) Hello(req *HelloReq) *HelloResp {
            return &HelloResp{
                Text: "hello " + req.Username + "!",
            }
        }
    
        type HelloReq struct {
            Username string `form:"username" binding:"required"`
        }
    
        type HelloResp struct {
            Text string `json:"text"`
        }
        ```
    * 修改 module.go 文件
        ```go
        package user

        import (
            "github.com/zeddy-go/zeddy/container"
            "github.com/zeddy-go/zeddy/contract"
            "github.com/zeddy-go/zeddy/module"
            "quickstart/module/user/iface/http"
        )
   
        func NewModule() *Module {
            m := &Module{}
            return m
        }
   
        type Module struct {
            module.BaseModule
        }
   
        func (m Module) Init() (err error) {
            err = container.Bind[*http.UserHandler](http.NewUserHandler)
            if err != nil {
                return
            }
            return
        }
   
        func (m Module) Boot() (err error) {
            err = container.Invoke(func(r contract.IRouter, userHandler *http.UserHandler) {
                r.GET("/hello", userHandler.Hello)
            })
            if err != nil {
                return
            }
            return
        }
        ```
4. 修改 main.go
    ```go
    package main
    
    import (
    	"github.com/zeddy-go/zeddy/app"
    	"github.com/zeddy-go/zeddy/http/ginx"
    	"log/slog"
    	"quickstart/module/user"
    )
    
    func main() {
    	app.Use(
    		ginx.NewModule(),
    		user.NewModule(),
    	)
    	err := app.StartAndWait()
    	if err != nil {
    		slog.Warn("An error occurred", "error", err)
    	}
    }
    ```
5. 再跑起来,然后访问 http://localhost:8080/hello?username=zed
    ```bash
    go run .
    ```
> 完整代码:
> ```bash
> git clone -b step1 --single-branch git@github.com:zeddy-go/quickstart.git
> ```
