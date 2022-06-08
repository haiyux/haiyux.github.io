---
title: "通过 layout 探索 kratos 运行原理"
date: 2021-10-02T18:03:24+08:00
draft: false
toc: true
categories: [microservice,go]
authors:
    - haiyux
---



## 创建项目

首先需要安装好对应的依赖环境，以及工具：

1. go  
   - [下载](https://studygolang.com/dl)
2. protoc 
   - `go install google.golang.org/protobuf/cmd/protoc-gen-go@latest`
3. protoc-gen-go
   - `go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest`

```bash
# 创建项目模板
kratos new helloworld

cd helloworld
# 拉取项目依赖
go mod download
# 生成proto模板
kratos proto add api/helloworld/helloworld.proto
# 生成proto源码
kratos proto client api/helloworld/helloworld.proto
# 生成server模板
kratos proto server api/helloworld/helloworld.proto -t internal/service
```

执行命令后,会在当前目录下生成一个 service 工程,工程骨架如下,具体的工程骨架说明可以访问 [layout](https://go-kratos.dev/docs/intro/layout) 

![](/images/2344773-20210902224825993-166434377.png)

## 运行项目

```shell
# 生成所有proto源码、wire等等
go generate ./...

# 编译成可执行文件
go build -o ./bin/ ./...

# 运行项目
./bin/helloworld -conf ./configs
```

看到如下输出则证明项目启动正常

```bash
level=INFO module=app service_id=7114ad8a-b3bf-11eb-a1b9-f0189850d2cb service_name=  version=
level=INFO module=transport/grpc msg=[gRPC] server listening on: [::]:9000
level=INFO module=transport/http msg=[HTTP] server listening on: [::]:8000 
```

测试接口

```shell
curl 'http://127.0.0.1:8000/helloworld/krtaos'

输出：
{
  "message": "Hello kratos"
}
```

## 应用是如何跑起来的?

![](/images/2344773-20210902224841022-2051641947.png)

通过上面的图例👆,我们可以直观观察到应用的调用链,简化来说如下图流程所示👇

![](/images/2344773-20210902224857367-1591524951.png)

### 1. 注入依赖并调用 newApp() 方法

```go
// helloword/cmd/main.go
func main() {
    flag.Parse()
    logger := log.NewStdLogger(os.Stdout)

    // 调用 go-kratos/kratos/v2/config,创建 config 实例,并指定了来源和配置解析方法
    c := config.New(
    config.WithSource(
        file.NewSource(flagconf),
    ),
    config.WithDecoder(func(kv *config.KeyValue, v map[string]interface{}) error {
        return yaml.Unmarshal(kv.Value, v)
    }),
    )
    if err := c.Load(); err != nil {
        panic(err)
    }

    // 将配置扫描到,通过 proto 声明的 conf struct 上
    var bc conf.Bootstrap
    if err := c.Scan(&bc); err != nil {
        panic(err)
    }

    // 通过 wire 将依赖注入,并调用 newApp 方法
    app, cleanup, err := initApp(bc.Server, bc.Data, logger)
    if err != nil {
        panic(err)
    }
    // 省略代码...
}
```

### 2. 创建 kratos 实例

项目 main.go 的 **newApp()** 方法中,调用了 **go-kratos/kratos/v2/app.go** 中的 **kratos.New()** 方法

```go
// helloword/cmd/main.go
func newApp(logger log.Logger, hs *http.Server, gs *grpc.Server) *kratos.App {
    return kratos.New(
        // 配置应用   
        kratos.Name(Name),
        kratos.Version(Version),
        kratos.Metadata(map[string]string{}),
        kratos.Logger(logger),
        // kratos.Server() 传入的 http/grpc 服务会通过 buildInstance() 转换成registry.ServiceInstance struct*
        kratos.Server(
            hs,
            gs,
        ),
    )
}
```

该方法会返回一个 **App struct**,包含 **Run()** 和 **Stop()** 方法

```go
// go-kratos/kratos/v2/app.go
type App struct {
    opts     options //配置
    ctx      context.Context // 上下文
    cancel   func() // context 的取消方法
    instance *registry.ServiceInstance //通过 kratos.Server()声明的实例,并通过 buildInstance() 转换后的 *registry.ServiceInstance struct
    log      *log.Helper // 日志
}

// Run executes all OnStart hooks registered with the application's Lifecycle.
func (a *App) Run() error {
    // 省略代码...
}

// Stop gracefully stops the application.
func (a *App) Stop() error {
    // 省略代码...
}
```

### 3. 调用 Run() 方法[#](https://go-kratos.dev/blog/go-layout-operation-process/#3-调用-run-方法)

项目在 main 方法中调用了 **kratos.App struct** 的 **Run()** 方法.

```go
// helloword/cmd/main.go
// 省略代码...
// 启动 Kratos
if err := app.Run(); err != nil {
    panic(err)
}
```

**Run()** 方法的实现细节

```go
// go-kratos/kratos/v2/app.go
func (a *App) Run() error {
    a.log.Infow(
        "service_id", a.opts.id,
        "service_name", a.opts.name,
        "version", a.opts.version,
    )
    g, ctx := errgroup.WithContext(a.ctx)
        // 遍历通过 kratos.Server() 声明的服务实例
    for _, srv := range a.opts.servers {
        srv := srv
                // 执行两个goroutine, 用于处理服务启动和退出
        g.Go(func() error {
            <-ctx.Done() // 阻塞,等待调用 cancel 方法
            return srv.Stop() // 协程退出后,调用实例的停止方法
        })
        g.Go(func() error {
            return srv.Start() // 调用实例的运行方法
        })
    }
        // 判断是否调用 kratos.Registrar() 配置了注册发现中心
    if a.opts.registrar != nil {
        // 将实例注册到注册中心
        if err := a.opts.registrar.Register(a.opts.ctx, a.instance); err != nil 
            return err
        }
    }
        // 监听进程退出信号
    c := make(chan os.Signal, 1)
    signal.Notify(c, a.opts.sigs...)
        
        // 处理进程退出和 context 退出
    g.Go(func() error {
        for {
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-c:
                        // 调用 kratos.App 的停止方法
                a.Stop()
            }
        }
    })
    if err := g.Wait(); err != nil && !errors.Is(err, context.Canceled) {
        return err
    }
    return nil
}
```

### 4. 应用退出

Kratos 实例在启动时,监听了系统的进程退出信号,当收到退出信号时,kratos 会调用 **App struct** 的 **Stop()** 方法

```go
// go-kratos/kratos/v2/app.go
func (a *App) Stop() error {
    // 判断是否有注册中心配置
    if a.opts.registrar != nil {
        // 在注册中心中将实例注销
        if err := a.opts.registrar.Deregister(a.opts.ctx, a.instance); err != nil {
            return err
        }
    }
    // 控制 goroutine 的退出,当调用 a.cancel()时,Run()方法中 监听的 <-ctx.Done() 收到消息后,没有阻塞后,方法会调用 server 的 Stop()方法,停止服务
    if a.cancel != nil {
        a.cancel()
    }
    return nil
}
```

## 文章转自：
- https://go-kratos.dev/blog/go-layout-operation-process/

---

![](/images/2344773-20210902225224456-315933124.png)

![](/images/2344773-20210902225203602-1750987546.gif)
