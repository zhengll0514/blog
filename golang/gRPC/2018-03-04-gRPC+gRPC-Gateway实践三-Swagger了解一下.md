# Swagger了解一下

在[上一节](https://segmentfault.com/a/1190000013408485)，我们完成了一个服务端同时支持`Rpc`和`RESTful Api`后，你以为自己大功告成了，结果突然发现要写`Api`文档和前端同事对接= = 。。。

你寻思有没有什么组件能够自动化生成`Api`文档来解决这个问题，就在这时你发现了`Swagger`，一起了解一下吧！

## 介绍
### Swagger
`Swagger`是全球最大的`OpenAPI`规范（OAS）API开发工具框架，支持从设计和文档到测试和部署的整个API生命周期的开发

`Swagger`是目前最受欢迎的`RESTful Api`文档生成工具之一，主要的原因如下
- 跨平台、跨语言的支持
- 强大的社区
- 生态圈 Swagger Tools（[Swagger Editor](https://github.com/swagger-api/swagger-editor)、[Swagger Codegen](https://github.com/swagger-api/swagger-codegen)、[Swagger UI](https://github.com/swagger-api/swagger-ui) ...）
- 强大的控制台

同时`grpc-gateway`也支持`Swagger`

[image]

### `OpenAPI`规范
`OpenAPI`规范是`Linux`基金会的一个项目，试图通过定义一种用来描述API格式或API定义的语言，来规范`RESTful`服务开发过程。`OpenAPI`规范帮助我们描述一个API的基本信息，比如：
- 有关该API的一般性描述
- 可用路径（/资源）
- 在每个路径上的可用操作（获取/提交...）
- 每个操作的输入/输出格式

目前V2.0版本的[OpenAPI规范](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md)（也就是SwaggerV2.0规范）已经发布并开源在github上。该文档写的非常好，结构清晰，方便随时查阅。

注：`OpenAPI`规范的介绍引用自[原文](https://huangwenchao.gitbooks.io/swagger/content/)

## 使用

### 生成`Swagger`的说明文件

**第一**，我们需要检查$GOBIN下是否包含`protoc-gen-swagger`可执行文件

若不存在则需要执行：
```
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger
```
等待执行完毕后，可在`$GOPATH/bin`下发现该执行文件，将其移动到`$GOBIN`下即可

**第二**，回到`$GOPATH/src/grpc-hello-world/proto`下，执行命令
```
protoc -I/usr/local/include -I. -I$GOPATH/src/grpc-hello-world/proto/google/api --swagger_out=logtostderr=true:. ./hello.proto
```
成功后执行`ls`即可看到`hello.swagger.json`文件


### 下载`Swagger UI`文件
`Swagger`提供可视化的`API`管理平台，就是[Swagger UI](https://github.com/swagger-api/swagger-ui)

我们将其源码下载下来，并将其`dist`目录下的所有文件拷贝到我们项目中的`$GOPATH/src/grpc-hello-world/third_party/swagger-ui`去

### 将`Swagger UI`转换为`Go`源代码

在这里我们使用的转换工具是[go-bindata](https://github.com/jteeuwen/go-bindata)

它支持将任何文件转换为可管理的`Go`源代码。用于将二进制数据嵌入到`Go`程序中。并且在将文件数据转换为原始字节片之前，可以选择压缩文件数据

#### 安装
```
go get -u github.com/jteeuwen/go-bindata/...
```
完成后，将`$GOPATH/bin`下的`go-bindata`移动到`$GOBIN`下

#### 转换
在项目下新建`pkg/ui/data/swagger`目录，回到`$GOPATH/src/grpc-hello-world/third_party/swagger-ui`下，执行命令
```
go-bindata --nocompress -pkg swagger -o pkg/ui/data/swagger/datafile.go third_party/swagger-ui/...
```

#### 检查
回到`pkg/ui/data/swagger`目录，检查是否存在`datafile.go`文件

### `Swagger UI`文件服务器（对外提供服务）
在这一步，我们需要使用与其配套的[go-bindata-assetfs](https://github.com/elazarl/go-bindata-assetfs/)

它能够使用`go-bindata`所生成`Swagger UI`的`Go`代码，结合`net/http`对外提供服务

#### 安装
```
go get github.com/elazarl/go-bindata-assetfs/...
```

#### 编写

通过分析，我们得知生成的文件提供了一个`assetFS`函数，该函数返回一个封装了嵌入文件的`http.Filesystem`，可以用其来提供一个`HTTP`服务

那么我们来编写`Swagger UI`的代码吧，主要是两个部分，一个是`swagger.json`，另外一个是`swagger-ui`的响应

##### serveSwaggerFile
引用包`strings`、`path`
```
func serveSwaggerFile(w http.ResponseWriter, r *http.Request) {
      if ! strings.HasSuffix(r.URL.Path, "swagger.json") {
        log.Printf("Not Found: %s", r.URL.Path)
        http.NotFound(w, r)
        return
    }

    p := strings.TrimPrefix(r.URL.Path, "/swagger/")
    p = path.Join("proto", p)

    log.Printf("Serving swagger-file: %s", p)

    http.ServeFile(w, r, p)
}
```
在函数中，我们利用`r.URL.Path`进行路径后缀判断

主要做了对`swagger.json`的文件访问支持（提供`https://127.0.0.1:50052/swagger/hello.swagger.json`的访问）

##### serveSwaggerUI
引用包`github.com/elazarl/go-bindata-assetfs`、`grpc-hello-world/pkg/ui/data/swagger`
```
func serveSwaggerUI(mux *http.ServeMux) {
    fileServer := http.FileServer(&assetfs.AssetFS{
        Asset:    swagger.Asset,
        AssetDir: swagger.AssetDir,
        Prefix:   "third_party/swagger-ui",
    })
    prefix := "/swagger-ui/"
    mux.Handle(prefix, http.StripPrefix(prefix, fileServer))
}
```

在函数中，我们使用了[go-bindata-assetfs](https://github.com/elazarl/go-bindata-assetfs/)来调度先前生成的`datafile.go`，结合`net/http`来对外提供`swagger-ui`的服务

#### 结合
在完成功能后，我们发现`path.Join("proto", p)`是写死参数的，这样显然不对，我们应该将其导出成外部参数，那么我们来最终改造一番

首先我们在`server.go`新增包全局变量`SwaggerDir`，修改`cmd/server.go`文件：
```
package cmd

import (
	"log"

	"github.com/spf13/cobra"
	
	"grpc-hello-world/server"
)

var serverCmd = &cobra.Command{
	Use:   "server",
	Short: "Run the gRPC hello-world server",
	Run: func(cmd *cobra.Command, args []string) {
		defer func() {
			if err := recover(); err != nil {
				log.Println("Recover error : %v", err)
			}
		}()
		
		server.Run()
	},
}

func init() {
	serverCmd.Flags().StringVarP(&server.ServerPort, "port", "p", "50052", "server port")
	serverCmd.Flags().StringVarP(&server.CertPemPath, "cert-pem", "", "./conf/certs/server.pem", "cert-pem path")
	serverCmd.Flags().StringVarP(&server.CertKeyPath, "cert-key", "", "./conf/certs/server.key", "cert-key path")
	serverCmd.Flags().StringVarP(&server.CertServerName, "cert-server-name", "", "grpc server name", "server's hostname")
	serverCmd.Flags().StringVarP(&server.SwaggerDir, "swagger-dir", "", "proto", "path to the directory which contains swagger definitions")
	
	rootCmd.AddCommand(serverCmd)
}
```

修改`path.Join("proto", p)`为`path.Join(SwaggerDir, p)`，这样的话我们`swagger.json`的文件路径就可以根据外部情况去修改它

最终`server.go`文件内容：
```
package server

import (
    "crypto/tls"
    "net"
    "net/http"
    "log"
    "strings"
    "path"

    "golang.org/x/net/context"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
    "github.com/grpc-ecosystem/grpc-gateway/runtime"
    "github.com/elazarl/go-bindata-assetfs"
    
    pb "grpc-hello-world/proto"
    "grpc-hello-world/pkg/util"
    "grpc-hello-world/pkg/ui/data/swagger"
)

var (
    ServerPort string
    CertServerName string
    CertPemPath string
    CertKeyPath string
    SwaggerDir string
    EndPoint string

    tlsConfig *tls.Config
)

func Run() (err error) {
    EndPoint = ":" + ServerPort
    tlsConfig = util.GetTLSConfig(CertPemPath, CertKeyPath)

    conn, err := net.Listen("tcp", EndPoint)
    if err != nil {
        log.Printf("TCP Listen err:%v\n", err)
    }

    srv := newServer(conn)

    log.Printf("gRPC and https listen on: %s\n", ServerPort)

    if err = srv.Serve(util.NewTLSListener(conn, tlsConfig)); err != nil {
        log.Printf("ListenAndServe: %v\n", err)
    }

    return err
}
 
func newServer(conn net.Listener) (*http.Server) {
    grpcServer := newGrpc()
    gwmux, err := newGateway()
    if err != nil {
        panic(err)
    }

    mux := http.NewServeMux()
    mux.Handle("/", gwmux)
    mux.HandleFunc("/swagger/", serveSwaggerFile)
    serveSwaggerUI(mux)

    return &http.Server{
        Addr:      EndPoint,
        Handler:   util.GrpcHandlerFunc(grpcServer, mux),
        TLSConfig: tlsConfig,
    }
}

func newGrpc() *grpc.Server {
    creds, err := credentials.NewServerTLSFromFile(CertPemPath, CertKeyPath)
    if err != nil {
        panic(err)
    }

    opts := []grpc.ServerOption{
        grpc.Creds(creds),
    }
    server := grpc.NewServer(opts...)

    pb.RegisterHelloWorldServer(server, NewHelloService())

    return server
}

func newGateway() (http.Handler, error) {
    ctx := context.Background()
    dcreds, err := credentials.NewClientTLSFromFile(CertPemPath, CertServerName)
    if err != nil {
        return nil, err
    }
    dopts := []grpc.DialOption{grpc.WithTransportCredentials(dcreds)}
    
    gwmux := runtime.NewServeMux()
    if err := pb.RegisterHelloWorldHandlerFromEndpoint(ctx, gwmux, EndPoint, dopts); err != nil {
        return nil, err
    }

    return gwmux, nil
}

func serveSwaggerFile(w http.ResponseWriter, r *http.Request) {
      if ! strings.HasSuffix(r.URL.Path, "swagger.json") {
        log.Printf("Not Found: %s", r.URL.Path)
        http.NotFound(w, r)
        return
    }

    p := strings.TrimPrefix(r.URL.Path, "/swagger/")
    p = path.Join(SwaggerDir, p)

    log.Printf("Serving swagger-file: %s", p)

    http.ServeFile(w, r, p)
}

func serveSwaggerUI(mux *http.ServeMux) {
    fileServer := http.FileServer(&assetfs.AssetFS{
        Asset:    swagger.Asset,
        AssetDir: swagger.AssetDir,
        Prefix:   "third_party/swagger-ui",
    })
    prefix := "/swagger-ui/"
    mux.Handle(prefix, http.StripPrefix(prefix, fileServer))
}
```

## 测试
访问路径`https://127.0.0.1:50052/swagger/hello.swagger.json`，查看输出内容是否为`hello.swagger.json`的内容，例如：
[image]

访问路径`https://127.0.0.1:50052/swagger-ui/`，查看内容
[image]

## 小结

至此我们这一章节就完毕了，`Swagger`和其生态圈十分的丰富，有兴趣研究的小伙伴可以到其[官网](https://swagger.io/)认真研究

而目前完成的程度也满足了日常工作的需求了，可较自动化的生成`RESTful Api`文档，完成与接口对接


## 参考
### 示例代码
- [grpc-hello-world](https://github.com/EDDYCJY/grpc-hello-world)

