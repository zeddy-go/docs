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
    * 修改 module/user/module.go 文件
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
### 数据存储
1. 启动数据库

   我们使用docker启动数据库，您也可以使用其他方式。
   
   新建 docker-compose.yaml 文件:
   ```yaml
   version: "3"
   services:
       mysql:
           image: "mysql:8.2"
           ports:
               - "3306:3306"
           volumes:
               - ./data/mysql:/var/lib/mysql
           environment:
               - MYSQL_ROOT_PASSWORD=toor
               - MYSQL_DATABASE=main
           networks:
               - app
   networks:
       app:
   ```
   执行命令:
   ```bash
   docker-compose up -d
   ```
2. 安装迁移工具和orm
   ```bash
   go install -tags 'mysql' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
   go get -u gorm.io/gorm
   ```
3. 创建迁移
   * 创建文件 module/user/infra/migration/migration.go
      ```go
      package migration

      import (
          "embed"
          "github.com/zeddy-go/zeddy/database/migrate"
      )
      
      //go:embed *.sql
      var migrateFs embed.FS
      
      func RegisterMigration(driver *migrate.EmbedDriver) {
          driver.Add(migrateFs)
      }
      ```
   * 创建迁移文件
      ```bash
      cd module/user/infra/migration && migrate create -ext sql create_users_table && cd ../../../../
      ```
   * 复制下面代码到 xxxxxxxx_create_users_table.up.sql 文件中
      ```mysql
      CREATE TABLE users (
         `id` BIGINT UNSIGNED PRIMARY KEY,
         `username` VARCHAR(255) NOT NULL DEFAULT '' COMMENT '用户名',
         `password` VARCHAR(255) NOT NULL DEFAULT '' COMMENT '密码',
         `created_at` BIGINT UNSIGNED NOT NULL DEFAULT 0,
         `updated_at` BIGINT UNSIGNED NOT NULL DEFAULT 0
      ) COMMENT='用户表' ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
      ```
   * 复制下面代码到 xxxxxxxx_create_users_table.down.sql 文件中
      ```mysql
      DROP TABLE users;
      ```
   * 在 module/user/module.go 文件中的 Init 方法中添加代码
      ```go
      package user

      import (
          "github.com/zeddy-go/zeddy/container"
          "github.com/zeddy-go/zeddy/contract"
          "github.com/zeddy-go/zeddy/module"
          "quickstart/module/user/iface/http"
          "quickstart/module/user/infra/migration" //+
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
         err = container.Invoke(migration.RegisterMigration) //+
         if err != nil {                                     //+
            return                                           //+
         }                                                   //+
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
4. 配置
   * 新建文件 config/config.go 复制以下代码
      ```go
      package conf

      import _ "embed"

      //go:embed config.yaml
      var Config string
      ```
   * 新建文件 config/config.yaml 复制以下代码
      ```yaml
      database:
          dsn: mysql://root:toor@tcp(mysql)/main?collation=utf8mb4_general_ci&loc=Asia/Shanghai&parseTime=true
      ```
5. 修改 main.go 文件
      ```go
      package main

      import (
          "github.com/zeddy-go/zeddy/app"
          "github.com/zeddy-go/zeddy/config"            //+
          "github.com/zeddy-go/zeddy/database/migrate"  //+
          "github.com/zeddy-go/zeddy/database/wgorm"    //+
          "github.com/zeddy-go/zeddy/http/ginx"
          "log/slog"
          conf "quickstart/config"                      //+
          "quickstart/module/user"
      )

      func main() {
          app.Use(
              config.NewModule(config.WithContent(conf.Config)), //+
              wgorm.NewModule(),                                 //+
              migrate.NewModule(),                               //+
              ginx.NewModule(),
              user.NewModule(),
          )
          err := app.StartAndWait()
          if err != nil {
              slog.Warn("An error occurred", "error", err)
          }
      }
      ```
   此时启动程序，你会看到数据库表已经自动创建到数据库了
6. 编写实体,模型和仓库
    * 新建 module/user/infra/model/user.go 文件
        ```go
        package model

        import "github.com/zeddy-go/zeddy/database/wgorm"

        type User struct {
            wgorm.CommonField
            Username string
            Password string
        }
        ```
    * 新建 module/user/domain/user.go 文件
        ```go
        package domain

        type User struct {
            ID       uint64
            Username string
            Password string
        }
        ```
    * 新建 module/user/domain/contract.go 文件
        ```go
        package domain

        import (
            "github.com/zeddy-go/zeddy/database"
            "gorm.io/gorm"
        )
        
        type IUserRepo interface {
            database.IRepository[User, *gorm.DB]
        }
        ```
    * 新建 module/user/infra/repo/user.go 文件
        ```go
        package repo

        import (
            "github.com/zeddy-go/zeddy/database"
            "github.com/zeddy-go/zeddy/database/wgorm"
            "gorm.io/gorm"
            "quickstart/module/user/domain"
            "quickstart/module/user/infra/model"
        )
        
        func NewUserRepo(db *gorm.DB) domain.IUserRepo {
            r := &UserRepository{}
            r.IRepository = wgorm.NewRepository[model.User, domain.User](db)
            return r
        }
        
        type UserRepository struct {
            database.IRepository[domain.User, *gorm.DB]
        }
        ```
7. 修改 module/user/module.go 文件
    ```go
    package user

    import (
        "github.com/zeddy-go/zeddy/container"
        "github.com/zeddy-go/zeddy/contract"
        "github.com/zeddy-go/zeddy/module"
        "quickstart/module/user/domain"          //+
        "quickstart/module/user/iface/http"
        "quickstart/module/user/infra/migration"
        "quickstart/module/user/infra/repo"      //+
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

        err = container.Invoke(migration.RegisterMigration)
        if err != nil {
            return
        }

        err = container.Bind[domain.IUserRepo](repo.NewUserRepo)  //+
        if err != nil {                                           //+
            return                                                //+
        }                                                         //+
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
8. 编写接口
    * 修改 module/user/iface/http/user.go 
        ```go
        package http

        import (
            "github.com/zeddy-go/zeddy/database"
            "quickstart/module/user/domain"
        )

        func NewUserHandler(userRepo domain.IUserRepo) *UserHandler {
            return &UserHandler{
                userRepo: userRepo,
            }
        }

        type UserHandler struct {
            userRepo domain.IUserRepo
        }

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

        func (uh *UserHandler) Create(req *CreateReq) (resp *CreateResp, err error) {
            user := &domain.User{
                Username: req.Username,
                Password: req.Password,
            }
            err = uh.userRepo.Create(user)
            if err != nil {
                return
            }
            resp = &CreateResp{
                ID: user.ID,
            }
            return
        }

        type CreateReq struct {
            Username string `json:"username" binding:"required"`
            Password string `json:"password" binding:"required"`
        }

        type CreateResp struct {
            ID uint64 `json:"id,string"`
        }

        func (uh *UserHandler) Detail(req *DetailReq) (resp *DetailResp, err error) {
            return uh.userRepo.First(database.Condition{"id", req.ID})
        }

        type DetailReq struct {
            ID uint64 `uri:"id" binding:"required"`
        }

        type DetailResp = domain.User
        ```
    * 修改 module/user/module.go 文件
       ```go
       package user

       import (
           "github.com/zeddy-go/zeddy/container"
           "github.com/zeddy-go/zeddy/contract"
           "github.com/zeddy-go/zeddy/module"
           "quickstart/module/user/domain"
           "quickstart/module/user/iface/http"
           "quickstart/module/user/infra/migration"
           "quickstart/module/user/infra/repo"
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

           err = container.Invoke(migration.RegisterMigration)
           if err != nil {
               return
           }

           err = container.Bind[domain.IUserRepo](repo.NewUserRepo)
           if err != nil {
               return
           }

           return
       }

       func (m Module) Boot() (err error) {
           err = container.Invoke(func(r contract.IRouter, userHandler *http.UserHandler) {
               r.GET("/hello", userHandler.Hello)
               r.POST("/api/users", userHandler.Create)    //+
               r.GET("/api/users/:id", userHandler.Detail) //+
           })
           if err != nil {
               return
           }
           return
       }
       ```
此时运行程序，通过调试工具调用相关接口，程序正常工作。
> 完整代码:
> ```bash
> git clone -b step2 --single-branch git@github.com:zeddy-go/quickstart.git
> ```