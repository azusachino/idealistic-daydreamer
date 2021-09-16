---
title: Protobuf Simple Practice
description: 我懂了(完全没懂)
date: 2021-09-13
slug: other/grpc-protobuf
image: img/2021/09/WalhallaOverlook.jpg
categories:
  - exp
---

鉴于最近接触到很多使用了 `grpc` 的服务，如果还不了解的话，就根本没办法继续学习下去了，所以(又)花了小半天时间再次熟悉了一下。

## protobuffer 安装

### 安装 proto-compiler

推荐使用 `chocolatey`，可以一键安装 `choco install protoc`

### 安装 ptotoc-gen-xx

可以从官方 github 的 release 下载, 如 [v3.17.3](https://github.com/protocolbuffers/protobuf/releases/tag/v3.17.3)，然后将 `protobuf-3.17.3/src/google` 文件夹下的内容保存到 `C:ProgramData/chocolatey/lib/protoc/tools/include/google` 中即可。

## 项目结构

```bash
.
├── go.mod
├── go.sum
├── mail
│   ├── client_test.go
│   ├── makefile
│   └── server.go
└── proto
    └── mail
        ├── mail.pb.go
        ├── mail.proto
        └── mail_grpc.pb.go
```

### mail.proto

```protobuf
syntax = "proto3"; // 指定protobuf版本

package mall; // 指定默认包名

// 指定生成go文件的包名
option go_package = "github.com/AzusaChino/grpc_demo/proto";

service MailService {
    rpc SendMail(MailRequest) returns (MailResponse) {
    }
}

// 请求消息
message MailRequest {
    string mail = 1;
    string text = 2;
}
// 响应消息
message MailResponse {
    bool ok = 1;
}
```

### makefile

- `mail.pb.go`: 主要包括 `message` 的相关定义
- `mail_grpc.pb.go`: 主要包括 `service` 的相关定义

通过指定相对路径，使用 `make build` 命令即可生成这两个文件。

```makefile
# parse proto files
build:
    protoc --proto_path=../proto --go_out=../proto --go_opt=paths=source_relative \
 --go-grpc_out=../proto --go-grpc_opt=paths=source_relative ../proto/mail/mail.proto
```

### server.go

```go
package main

import (
    "context"
    "fmt"
    pb "github.com/AzusaChino/grpc_demo/proto/mail"
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
    "net"
)

type service struct {
    // fallback solution, service must implement
    pb.UnimplementedMailServiceServer
}

func (s *service) SendMail(ctx context.Context, req *pb.MailRequest) (res *pb.MailResponse, err error) {
    fmt.Printf("mail: %s, msg detail: %s\n", req.Mail, req.Text)
    return &pb.MailResponse{
        Ok: true,
    }, nil
}

func main() {
    // listen tcp 8972
    lis, err := net.Listen("tcp", ":8972")
    if err != nil {
        fmt.Printf("failed to listen: %v", err)
        return
    }
    // grpc server
    s := grpc.NewServer()
    pb.RegisterMailServiceServer(s, &service{})
    reflection.Register(s)

    if err := s.Serve(lis); err != nil {
        fmt.Println(err)
    }
}
```

### client_test.go

```go
package main

import (
    "context"
    pb "github.com/AzusaChino/grpc_demo/proto/mail"
    "google.golang.org/grpc"
    "log"
    "testing"
    "time"
)

func TestService_SendMail(t *testing.T) {

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    // dial grpc server
    conn, err := grpc.DialContext(ctx, "127.0.0.1:8972", grpc.WithInsecure(),
        grpc.WithBlock())
    if err != nil {
        log.Fatalf("failed to dial: %v", err)
    }

    c := pb.NewMailServiceClient(conn)

    res, err := c.SendMail(context.TODO(), &pb.MailRequest{
        Mail: "abc@protonmail.com",
        Text: "Hello",
    })
    log.Println(res)
}
```

## 暂时性总结

这是个十分简单的 demo，后续学习一下和 etcd 之间的交互。

## 参考链接

- [Protocol Buffers](https://developers.google.com/protocol-buffers/docs/reference/go-generated#package)
- [golang grpc 之 etcd 服务注册发现](https://www.jianshu.com/p/e1a809e72fb7)
