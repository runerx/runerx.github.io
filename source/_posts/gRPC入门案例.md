---
title: gRPC入门案例
tags:
  - Golang
  - Code
categories: Code
date: 2022-06-10 20:21:08
---


gRPC入门案例

<!-- more -->

## 1. 环境准备

### 1.1 golang配置

```
#Golang
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```



### 1.2 安装gRPC

```
go get -u google.golang.org/grpc
```





### 1.3 Protocol buffer安装

官网：https://developers.google.com/protocol-buffers

The protocol compiler is written in C++. If you are using C++, please follow the [C++ Installation Instructions](https://github.com/protocolbuffers/protobuf/blob/main/src/README.md) to install protoc along with the C++ runtime.

For non-C++ users, the simplest way to install the protocol compiler is to download a pre-built binary from our release page https://github.com/protocolbuffers/protobuf/releases



下载

```

wget https://github.com/protocolbuffers/protobuf/releases/download/v21.1/protoc-21.1-linux-x86_64.zip

unzip -d protoc-21.1 protoc-21.1-linux-x86_64.zip


mv protoc-21.1 /usr/local

```

添加环境变量

```
#PROTOBUF
export PROTOBUF_HOME=/usr/local/protoc-21.1
export PATH=$PROTOBUF_HOME/bin:$PATH
```

确认 protoc 安装完成。

```
[root@debian pkg]# protoc --version
libprotoc 3.21.1
```



### 1.4 gRPC in Go

https://grpc.io/docs/languages/go/quickstart/

```
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2

source ~/.zshrc
```

确认 protoc-gen-go 安装完成。

```
[root@debian pkg]# protoc-gen-go --version
protoc-gen-go v1.28.0
```

确认 protoc-gen-go-grpc 安装完成。

```
[root@debian pkg]# protoc-gen-go-grpc --version
protoc-gen-go-grpc 1.2.0
```



## 2. Demo测试

### 2.1 编写proto代码

1. 新建项目grpc-demo，根目录执行`go mod init grpc-demo`

2. 新建grpc-demo/helloword文件夹，编写helloword.proto

   - helloword.proto

   ```protobuf
   syntax = "proto3"; // 版本声明，使用Protocol Buffers v3版本
   
   option go_package = "gprc-demo/helloworld/helloworld";
   
   package helloworld; // 包名
   
   
   // The greeting service definition.
   service Greeter {
       // Sends a greeting
       rpc SayHello (HelloRequest) returns (HelloReply) {}
     }
     
     // The request message containing the user's name.
     message HelloRequest {
       string name = 1;
     }
     
     // The response message containing the greetings
     message HelloReply {
       string message = 1;
     }
   ```

   在项目根目录下执行

   ```
   protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       helloworld/helloworld.proto
   ```

   

### 2.2 编写Server代码

新建grpc-demo/greeter_server目录

- main.go

  ```go
  package main
  
  import (
  	"context"
  	"flag"
  	"fmt"
  	"log"
  	"net"
  
  	"google.golang.org/grpc"
  	pb "grpc-demo/helloworld"
  )
  
  var (
  	port = flag.Int("port", 50051, "The server port")
  )
  
  // server is used to implement helloworld.GreeterServer.
  type server struct {
  	pb.UnimplementedGreeterServer
  }
  
  // SayHello implements helloworld.GreeterServer
  func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
  	log.Printf("Received: %v", in.GetName())
  	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
  }
  
  func main() {
  	flag.Parse()
  	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
  	if err != nil {
  		log.Fatalf("failed to listen: %v", err)
  	}
  	s := grpc.NewServer()
  	pb.RegisterGreeterServer(s, &server{})
  	log.Printf("server listening at %v", lis.Addr())
  	if err := s.Serve(lis); err != nil {
  		log.Fatalf("failed to serve: %v", err)
  	}
  }
  ```

  运行

  ```
  [root@debian grpc-demo]# go run greeter_server/main.go
  2022/06/10 19:01:52 server listening at [::]:50051
  ```



### 2.3 编写Client代码

新建grpc-demo/greeter_client目录

- main.go

  ```go
  /*
   *
   * Copyright 2015 gRPC authors.
   *
   * Licensed under the Apache License, Version 2.0 (the "License");
   * you may not use this file except in compliance with the License.
   * You may obtain a copy of the License at
   *
   *     http://www.apache.org/licenses/LICENSE-2.0
   *
   * Unless required by applicable law or agreed to in writing, software
   * distributed under the License is distributed on an "AS IS" BASIS,
   * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   * See the License for the specific language governing permissions and
   * limitations under the License.
   *
   */
  
  // Package main implements a client for Greeter service.
  package main
  
  import (
  	"context"
  	"flag"
  	"log"
  	"time"
  
  	"google.golang.org/grpc"
  	"google.golang.org/grpc/credentials/insecure"
  	pb "grpc-demo/helloworld"
  )
  
  const (
  	defaultName = "world"
  )
  
  var (
  	addr = flag.String("addr", "localhost:50051", "the address to connect to")
  	name = flag.String("name", defaultName, "Name to greet")
  )
  
  func main() {
  	flag.Parse()
  	// Set up a connection to the server.
  	conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
  	if err != nil {
  		log.Fatalf("did not connect: %v", err)
  	}
  	defer conn.Close()
  	c := pb.NewGreeterClient(conn)
  
  	// Contact the server and print out its response.
  	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
  	defer cancel()
  	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name})
  	if err != nil {
  		log.Fatalf("could not greet: %v", err)
  	}
  	log.Printf("Greeting: %s", r.GetMessage())
  }
  ```

  运行

  ```
  [root@debian grpc-demo]# go run greeter_client/main.go --name=Alice
  2022/06/10 19:05:33 Greeting: Hello Alice
  ```



服务端收到了消息

```
[root@debian grpc-demo]# go run greeter_server/main.go
2022/06/10 19:01:52 server listening at [::]:50051
2022/06/10 19:05:33 Received: Alice
```





## 3. 参考

- https://grpc.io/docs/languages/go/quickstart/
- https://www.liwenzhou.com/posts/Go/gRPC/

