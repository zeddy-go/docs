# 快速开始

1. 初始化一个 `go module` 项目
2. 将下面的代码复制到main.go中

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
3. 跑起来

```bash
go run .
```

这时,程序会输出类似如下的信息,并且停止:
```shell
2023/12/29 16:42:03 INFO nothing started, shutdown.
```

第一个zeddy程序已经成功运行了~

> 如你所见, 程序仅仅输出了一条日志便退出了. 这是因为我们还没有往程序中添加 `模块`. zeddy 的目标是提供一个能够快速上手, 并且完全由你掌控的开发体验.
    所以一个初始的 zeddy 程序并不会附加任何功能.