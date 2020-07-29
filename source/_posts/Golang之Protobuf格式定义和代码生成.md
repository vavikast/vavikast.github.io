---
title: Golang之Protobuf格式定义和代码生成
catalog: true
date: 2020-07-29 16:43:53
subtitle:
header-img:
tags:
- golang
- protobuf
- grpc
catagories:
- golang
- protobuf
- grpc
updateDate: 2020-07-29 16:43:53
---

#### 1.Protobuf格式定义（首部）

- protobuf代码说明

  

  addresssbook.proto

  ```
  syntax = "proto3";  //指定proto为版本3
  package tutorial;  //包命名，确保不同项目的包名不冲突，导入其他proto文件会用到。
  
  import "google/protobuf/timestamp.proto";
  //导入其他目录下的proto包，个人理解这里的导入包的位置，为指定源目录的相对位置，即protoc -I=$SRC-DIR目录位置，如果没有指定，则为当前执行命令目录。
  
  option go_package = "github.com/protocolbuffers/protobuf/examples/go/tutorialpb";
  //指定生成go程序的位置和包名（不是包文件名）。最后路径的最后一位是目录名，也是go程序的包名，即packege tutorialpb。(不是go程序文件名，go程序文件名和proto文件同名，即为addresssbook.pb.go) 
  
  //以下是通用配置不做解释
  message Person {
    string name = 1;
    int32 id = 2;  // Unique ID number for this person.
    string email = 3;
  
    enum PhoneType {
      MOBILE = 0;
      HOME = 1;
      WORK = 2;
    }
  
    message PhoneNumber {
      string number = 1;
      PhoneType type = 2;
    }
  
    repeated PhoneNumber phones = 4;
  
    google.protobuf.Timestamp last_updated = 5;
  }
  
  // Our address book file is just one of these.
  message AddressBook {
    repeated Person people = 1;
  }
  
  ```

  

- protobuf组织结构

  此处以一个项目talkischeap为例。

  ​		运行程序时，一般在talkischeap目录下运行。

  基本文件目录结构：

  ```
  talkischeap
  	-| proto
  		-| addressboook.proto
  	-| google
  		-| protobuf
  			-| timestd.proto
  ```

  具体代码：

  addressbook.proto代码：

  ```
  // [START declaration]
  syntax = "proto3";
  package tutorial;  //注意包名是tutorial
  
  
  option go_package = "proto/tutorialtb"; //注意指定生成的go程序的包是package tutorialtb。和文件名无关，文件名是proto文件的相对名称。
  
  import "google/protobuf/timestd.proto";
  // 注意导入了相对位置在google/protobuf目录下的timestd.proto.
  
  message Person {
    string name = 1;
    int32 id = 2;  // Unique ID number for this person.
    string email = 3;
  
    enum PhoneType {
      MOBILE = 0;
      HOME = 1;
      WORK = 2;
    }
  
    message PhoneNumber {
      string number = 1;
      PhoneType type = 2;
    }
  
    repeated PhoneNumber phones = 4;
  
    google.protobuf.Timestamp last_updated = 5;
    //注意此处，引用的是google.protobuf包下的Timstamp，在timestd.proto包中。
    
  }
  
  // Our address book file is just one of these.
  message AddressBook {
    repeated Person people = 1;
  }
  // [END messages]
  ```

  ​	timestd.proto中的代码

  ```
  syntax = "proto3";
  
  package google.protobuf;  //注意包名是google.protobuf.
  
  option go_package = "proto/tutorialpb/timestamp";
  //注意指定生成的go程序的包是package timestamp。和文件名无关，文件名是proto文件的相对名称。
  
  
  message Timestamp {
    // Represents seconds of UTC time since Unix epoch
    // 1970-01-01T00:00:00Z. Must be from 0001-01-01T00:00:00Z to
    // 9999-12-31T23:59:59Z inclusive.
    int64 seconds = 1;
  
    // Non-negative fractions of a second at nanosecond resolution. Negative
    // second values with fractions must still have non-negative nanos values
    // that count forward in time. Must be from 0 to 999,999,999
    // inclusive.
    int32 nanos = 2;
  }
  ```

  最终生成的目录结构如下：

  ```
  talkischeap
  	-| proto
  		-| addressboook.proto
  	-| tutorialpb
  		-| timestamp
  			-| tiemstd.pb.go
  	-| tutorialtb
  		-| addressbook.pb.go
  	-| google
  		-| protobuf
  			-| timestd.proto
  ```

  ![image.png](https://i.loli.net/2020/07/29/YinfGJNPtgDQZhT.png)

- protobuf 几点理解

  总结几点：

  ​	 #proto文件名和生成go程序文件名呼应。timestamp.proto -> timestamp.pb.go

  ​	 #proto生成的go程序的包和option go_package中的最后一个文件名对应。

  ​		例如 option go_package = github.com/protocolbuffers/protobuf中生成包为package protobuf。

  ​	#proto中的packge name是用于区别不同proto文件，以及其他文件导入使用。例如上面的google.protobuf.Timestamp

  ​	#proto的文件位置很重要，特别要注意，如果import其他位置的包，则沿着相应的路径找到对应文件。

#### 2.golang代码编译支持

- 下载安装编译器

  [下载页面](https://github.com/protocolbuffers/protobuf/releases/tag/v3.12.4)

  将命令工具放在环境变量下，即变成可执行程序。

- 安装golang插件

  ```shell
  go install google.golang.org/protobuf/cmd/protoc-gen-go
  //可以通过go mod将代码快速拉下来，然后在相应的目录下运行。
  //将protoc-gen-go命令也放在环境变量下，即变成可执行程序。
  ```

- 运行编译器生成代码

  ```
  protoc -I=$SRC_DIR --go_out=plugins=grpc:. $SRC_DIR/addressbook.proto
  -I: 指定proto文件所在的源目录，如果不指定，则为运行程序的当前目录
  $SRC_DIR/addressbook.proto：指定了具体的文件，也可以通过通配符指定目录下所有.proto文件，即$SRC_DIR/*.proto
  --go_out=plugins=grpc:. 要指定plugins=grpc的模式运行
  ```

  

  这里有一个坑点:

  protoc-gen-go插件升级了，相关的执行命令也改变了。所以别下载错版本了。

  ​		

个人理解，仅供参考。



#### 参考链接：

[Protocol Buffer Basics: Go](https://developers.google.com/protocol-buffers/docs/gotutorial)

[A basic tutorial introduction to gRPC in Go](https://grpc.io/docs/languages/go/basics/)

[Go语言使用gRPC的入门指南](https://golangcode.top/2018/09/26/grpc-basics-go/)





