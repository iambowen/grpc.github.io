---
layout: docs
title: gRPC Basics - Objective-C
---

# gRPC 基础： Objective-C

本教程提供了 Objective-C 程序员如何使用 gRPC 的指南。通过学习教程中例子，你可以学会如何：

- 在一个 .proto 文件内定义服务.
- 用 protocol buffer 编译器生成客户端代码.
- 使用 gRPC 的 Objective-C API 为你的服务实现一个简单的客户端.

It assumes a passing familiarity with [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview). Note that the example in this tutorial uses the proto3 version of the protocol buffers language, which is currently in alpha release: you can find out more in the [proto3 language guide](https://developers.google.com/protocol-buffers/docs/proto3) and see the [release notes](https://github.com/google/protobuf/releases) for the new version in the protocol buffers Github repository.

这算不上是一个在 Objective-C 中使用 gRPC 的综合指南：以后会有更多的参考文档.

## 为什么使用 gRPC?

有了 gRPC， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端和服务器，反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑- gRPC 帮你解决了不同语言间通信的复杂性以及环境的不同.使用 protocol buffers 还能获得其他好处，包括高效的序列号，简单的 IDL 以及容易进行接口更新。

gRPC 和 proto3 特别适合移动客户端：gRPC 基于 HTTP/2 实现，相比 HTTP/1.1 更加节省网络带宽。序列化和解析 proto 的二进制格式效率高于 JSON，节省了 CPU 和 电池消耗。proto3 使用的运行时在 Google 以及被优化了多年，代码量极小。这对于 Objective-C 非常重要，因为语言的动态天性，编译器在优化不使用的代码时受到了限制。

## 例子的代码和设置

教程的代码在这里 [grpc/grpc/examples/objective-c/route_guide](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/objective-c/route_guide)。 要下载例子，通过运行下面的命令去克隆`grpc`代码库：

```
$ git clone https://github.com/grpc/grpc.git
$ cd grpc
$ git submodule update --init
```

然后改变当前的目录到 `examples/objective-c/route_guide`:

```
$ cd examples/objective-c/route_guide
```

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互路由信息，如服务器和其他客户端的流量更新。

你还需要安装 [Cocoapods](https://cocoapods.org/#install) 以及相关的生成客户端类库的工具（以及一个用其他语言实现的服务器，出于测试的目的）。你可以根据[这些设置指南](https://github.com/grpc/homebrew-grpc)来得到后面的内容。

## 来试试吧！

为了使用例子应用，我们需要本地运行一个 gRPC 的服务器。让我们来编译运行，比如这个代码库中的 C++ 服务器：

```
$ pushd ../../cpp/route_guide
$ make
$ ./route_guide_server &
$ popd
```

现在让 Cocoapods 为我们的 .proto 文件生成和安装客户端类库：

```
$ pod install
```

（这也许需要编译 OpenSSL, 如果电脑上没有 Cocoapods 的缓存，大概需要15分钟能够完成）。
最后，打开 Cocoapods 生成的 Xcode workspace，运行应用。你可以在 `ViewControllers.m` 中检查调用的代码，可以从 XCode 的日志窗口看到结果。

下面的部分会指导你一步步的理解 proto 服务如何定义，如何从中生成一个客户端类库，以及如何使用类库创建一个应用。

## 定义服务

首先来看看我们使用的服务是如何定义的。gRPC 的 *service* 和它的方法 *request* 以及 *response* 类型使用了[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview)。你可以在[`examples/protos/route_guide.proto`](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/protos/route_guide.proto)看到完整的 .proto 文件。

要定义一个服务，你必须在你的 .proto 文件中指定 `service`：

```protobuf
service RouteGuide {
   ...
}
```
然后在你的服务中定义 `rpc` 方法，指定请求的和响应类型。gRPC允 许你定义4种类型的 service 方法，在 `RouteGuide` 服务中都有使用：

- 一个 *简单 RPC* ， 客户端使用存根发送请求到服务器并等待响应返回，就像平常的函数调用一样。

```protobuf
   // Obtains the feature at a given position.
   rpc GetFeature(Point) returns (Feature) {}
```

- 一个 *应答流式 RPC* ， 客户端发送请求到服务器，拿到返回的应答消息流。通过在 *响应* 类型前插入 `stream` 关键字，可以指定一个服务器端的流方法。

```protobuf
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

- 一个 *请求流式 RPC* ， 客户端发送一个消息序列到服务器。一旦客户端完成写入消息，它等待服务器完成读取返回它的响应。通过在 *请求* 类型前指定 `stream` 关键字来指定一个客户端的流方法。

```protobuf
  // Accepts a stream of Points on a route being traversed, returning a
  // RouteSummary when traversal is completed.
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

- 一个 *双向流式 RPC* 是双方使用读写流去发送一个消息序列。两个流独立操作，因此客户端和服务器可以以任意喜欢的顺序读写：比如， 服务器可以在写入响应前等待接收所有的客户端消息，或者可以交替的读取和写入消息，或者其他读写的组合。 每个流中的消息顺序被预留。你可以通过在请求和响应前加 `stream` 关键字去制定方法的类型。

```protobuf
  // Accepts a stream of RouteNotes sent while a route is being traversed,
  // while receiving other RouteNotes (e.g. from other users).
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

我们的 .proto 文件也包含了所有请求的 protocol buffer 消息类型定义以及在服务方法中使用的响
应类型——比如，下面的`Point`消息类型：

```protobuf
// Points are represented as latitude-longitude pairs in the E7 representation
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

通过在文件开始处添加 `objc_class_prefix` 选项，你可以为生成的类指定一个前缀。比如：

```protobuf
option objc_class_prefix = "RTG";
```

## 生成客户端代码

接下来我们需要从 .proto 的服务定义中生成 gRPC 客户端接口。我们通过 protocol buffer 的编译器 `protoc` 以及一个特殊的 gRPC Objective-C 插件来完成。

简单起见，我们提供一个 [Podspec 文件](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/objective-c/route_guide/RouteGuide.podspec) 帮你用合适的插件，输入，输出以及描述如何编译生成的文件去运行 `protoc`。你只需要在(`examples/objective-c/route_guide`)目录下运行：

```
$ pod install
```

这样会在这个例子的 XCode 项目中安装类库之前，运行：

```
$ protoc -I ../../protos --objc_out=Pods/RouteGuide --objcgrpc_out=Pods/RouteGuide ../../protos/route_guide.proto
```

运行这个命令会在 `Pods/RouteGuide/` 目录下生成下面的文件：

- `RouteGuide.pbobjc.h`，声明生成的消息类的头文件。
- `RouteGuide.pbobjc.m`，包含你的消息类的实现。
- `RouteGuide.pbrpc.h`，声明生成的服务类的头文件。
- `RouteGuide.pbrpc.m`，包含了你的服务类的实现。

这些包括：

这些包括：
- 所有用于填充，序列化和获取我们请求和响应消息类型的 protocol buffer 代码
- 一个名为 `RTGRouteGuide` 的类，可以让客户端调用定义在 `RouteGuide` 服务中的方法。

你也可以使用提供的 Podspec 文件从任意其它的 proto 服务生成客户端代码；只需要替换名字（匹配文件名），版本以及其它metadata。

## 创建客户端应用

在这个部分，我们会使用 `RouteGuide` 服务去创建一个 Objective-C 客户端。在[examples/objective-c/route_guide/ViewControllers.m](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/objective-c/route_guide/ViewControllers.m)可以看到我们完整的客户端例子代码。（注意：在你的应用中，出于维护和可读的原因，你不应该将所有的view controller放在一个文件中；这里这么做只是为了简化学习过程）。

### 构造一个服务对象

要调用一个服务方法，我们首先需要创建一个服务对象，生成的 `RTGRouteGuide` 类的一个实例。该类的初始化期望一个带有服务器地址以及我们期望连接端口的 `NSString *` ：

```objective-c
#import <GRPCClient/GRPCCall+Tests.h>
#import <RouteGuide/RouteGuide.pbrpc.h>

static NSString * const kHostAddress = @"localhost:50051";

...

[GRPCCall useInsecureConnectionsForHost:kHostAddress];

RTGRouteGuide *service = [[RTGRouteGuide alloc] initWithHost:kHostAddress];
```

注意，在构造我们的服务对象前，我们被告知 gRPC 类库在使用不安全的连接到 host:port。这是因为用来测试我们客户端的服务器没有使用[TLS](http://en.wikipedia.org/wiki/Transport_Layer_Security)。这么做没什么关系因为服务器只在本地开发环境运行。虽然最常见的场景是通过互联网连接支持 TLS 的 gRPC 服务器。对于那种场景，就不需要 `useInsecureConnectionsForHost:` 调用，如果没有指明，端口缺省为443。

### 调用服务方法

现在让我们来看看如何调用服务方法。如你所见，所有的这些方法都是异步的，所以你可以在应用的主线程中调用他们，不用担心 UI 被冻结或者 OS 杀掉你的应用。

#### 简单 RPC

调用简单 RPC `GetFeature` 几乎是和调用 Cocoa 的任何异步方法一样直观。

```objective-c
RTGPoint *point = [RTGPoint message];
point.latitude = 40E7;
point.longitude = -74E7;

[service getFeatureWithRequest:point handler:^(RTGFeature *response, NSError *error) {
  if (response) {
    // Successful response received
  } else {
    // RPC error
  }
}];
```

如你所见，我们创建并且填充了一个请求的 protocol buffer 对象（例子中为 `RTGPoint`）。然后，我们调用了服务对象的方法，传入请求，处理应答（或者任何 RPC 错误）的块。如果 RPC 顺利完成，处理的块和一个 `nil` 错误参数被调用，我们可以从服务器从应答参数中读取应答信息。如果，相反的，发生了 RPC 错误，处理的块和一个 `nil` 错误参数被调用，我们可以从错误参数中读取到问题的细节。

```objective-c
NSLog(@"Found feature called %@ at %@.", response.name, response.location);
```

#### 流式 RPC

现在让我们看看流式方法。下面是我们调用的应答流方法 `ListFeatures`，我们的客户端应用之后会收到一个地理位置的 `RTGFeature` 流：

```objective-c
[service listFeaturesWithRequest:rectangle handler:^(BOOL done, RTGFeature *response, NSError *error) {
  if (response) {
    // Element of the stream of responses received
  } else if (error) {
    // RPC error; the stream is over.
  }
  if (done) {
    // The stream is over (all the responses were received, or an error occured). Do any cleanup.
  }
}];
```

注意处理块的签名现在包括了一个 `BOOL done` 的参数。处理块可以被随意调用；只有在最后一次调用后 `done` 参数会被设置为 `YES`。一旦有错误发生，RPC 结束，处理块和参数 `(YES, nil, error)` 一起被调用。

请求流方法 `RecordRoute` 期望从客户端发来的 `RTGPoint` 流。这个流以遵循 `GRXWriter` 协议的对象形式被传入方法中。创建流的最简单的办法就是从 `NSArray` 对象中初始化一个：


```objective-c
#import <gRPC/GRXWriter+Immediate.h>

...

RTGPoint *point1 = [RTGPoint message];
point.latitude = 40E7;
point.longitude = -74E7;

RTGPoint *point2 = [RTGPoint message];
point.latitude = 40E7;
point.longitude = -74E7;

GRXWriter *locationsWriter = [GRXWriter writerWithContainer:@[point1, point2]];

[service recordRouteWithRequestsWriter:locationsWriter handler:^(RTGRouteSummary *response, NSError *error) {
  if (response) {
    NSLog(@"Finished trip with %i points", response.pointCount);
    NSLog(@"Passed %i features", response.featureCount);
    NSLog(@"Travelled %i meters", response.distance);
    NSLog(@"It took %i seconds", response.elapsedTime);
  } else {
    NSLog(@"RPC error: %@", error);
  }
}];

```

`GRXWriter` 足够通用，可以允许异步流，feature 值流，甚至无限流。

最后，让我们看看双向流式 RPC `RouteChat()`。调用一个双向流式 RPC 的方式仅是如何调用请求流 RPC 和应答流 RPC 的组合。

```objective-c
[service routeChatWithRequestsWriter:notesWriter handler:^(BOOL done, RTGRouteNote *note, NSError *error) {
  if (note) {
    NSLog(@"Got message %@ at %@", note.message, note.location);
  } else if (error) {
    NSLog(@"RPC error: %@", error);
  }
  if (done) {
    NSLog(@"Chat ended.");
  }
}];
```

处理块的语义以及这里的 `GRXWriter` 参数和我们的请求流和应答流方法一致。虽然客户端和服务器获取对方信息的顺序和信息被写入的顺序一致，读写流的操作是完全独立的。
