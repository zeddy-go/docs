# configx

框架模块 configx 负责读取配置，供其他模块使用。
configx 底层使用 github.com/spf13/viper ，confix 将他设置为读取配置后再自动读取环境变量，这样环境变量将会覆盖已有配置项的值。

## 读取配置
在快速开始示例中，我们使用embed来将配置文件作为字符串嵌入到程序中(示例项目/conf)，这时相关代码如下：
```go
app.Use(
    configx.NewModule(configx.WithContent(conf.Config)),
)
```
configx.WithContent 方法接收一个字符串，这样在编译后，我们就得到了一个已经包含默认配置的程序。在部署时，我们只需要修改环境变量即可修改程序配置。

您也可以像以前一样，为每个程序附带一个配置文件，那么代码是下面这样：
```go
app.Use(
    configx.NewModule(configx.WithPath("/path/to/config.yaml")),
)
```
configx.WithPath 方法接收一个路径。同样的，环境变量也能覆盖已有的配置项。

## 使用环境变量表示层级关系
因环境变量名中无法使用 `.` 号，所以我们使用 `__`(两个下划线) 来替换 `.`。
例如，在配置文件中我们有这样的配置：
```yaml
database:
    dsn: "xxxxxxxxxx"
```

那么环境变量我们需要写成这样：

```bash
DATABASE__DSN=xxxxxxxxxxx
```

## 模式
configx 中自带了四种模式：
* local：本地开发模式
* develop: 线上开发环境模式
* staging：线上预发布环境模式
* release：线上正式环境模式

在处理某些任务的时候，框架会需要知道当前的模式。

## slog
框架默认使用 slog 包作为日志输出工具, 在 configx 初始化时会顺便配置 slog 包。
若当前为 local 模式，configx 包会将 slog 的最低打印 level 设置为 debug。