# IoC/DI容器
IoC/DI 已经不是一个新概念了，这里不再赘述。
框架的核心便是一个IoC/DI容器(github.com/zeddy-go/zeddy/container)。
框架运行时，通过包级函数与包中唯一一个容器实例交互。
下面是一个例子：
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
