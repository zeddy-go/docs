# IoC/DI容器
在示例项目中您能看到 module.go 文件中使用了 github.com/zeddy-go/zeddy/container 包。
它是框架的核心，框架和其他代码都围绕着它运作。
IoC/DI 的概念，这里不再赘述。它的用法也是非常简单。
下面是两个例子：
```go
    type A struct{}

    type B struct {
        *A
    }

    _ = container.Bind[*A](func() *A {
        return &A{}
    })

    _ = container.Bind[*B](func(a *A) *B {
        return &B{
            A: a,
        }
    })

    //操作1
    b, _ := container.Resolve[*B]()
    fmt.Printf("%+v\n", b)

    //操作2
    container.Invoke(func(b *B) {
        fmt.Printf("%+v\n", b)
    })
```
操作1的效果等于操作2的效果。

Bind函数默认以单件模式绑定对象，若需以非单件模式绑定可以使用 `container.NoSingleton` 选项：
```go
    container.Bind[*A](func() *A {
        return &A{}
    }, container.NoSingleton())
```
