---
layout: docs
title: gRPC Basics - Python
---

# gRPC 基础: Python


本教程提供了 Python 程序员如何使用 gRPC 的指南。

通过学习教程中例子，你可以学会如何：

- 在一个 .proto 文件内定义服务.
- 用 protocol buffer 编译器生成服务器和客户端代码.
- 使用 gRPC 的 Python API 为你的服务实现一个简单的客户端和服务器.

假设你已经阅读了[概览](/docs/index.html)并且熟悉[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview). 注意，教程中的例子使用的是 protocol buffers 语言的 proto3 版本，它目前只是 alpha 版：可以在[ proto3 语言指南](https://developers.google.com/protocol-buffers/docs/proto3)和 protocol buffers 的 Github 仓库的[版本注释](https://github.com/google/protobuf/releases)发现更多关于新版本的内容.

这算不上是一个在 Python 中使用 gRPC 的综合指南：以后会有更多的参考文档.

## 为什么使用 gRPC?

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互路由信息，如服务器和其他客户端的流量更新。

有了 gRPC， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端
和服务器，反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑- gRPC 帮你解决了
不同语言间通信的复杂性以及环境的不同.使用 protocol buffers 还能获得其他好处，包括高效的序
列号，简单的 IDL 以及容易进行接口更新。

## 例子代码和设置

教程的代码在这里 [grpc/grpc/examples/python/route_guide](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/python/route_guide)。 要下载例子，通过运行下面的命令去克隆`grpc`代码库：

```
$ git clone https://github.com/grpc/grpc.git
```

改变当前的目录到 `examples/python/route_guide`：

```
$ cd examples/python/route_guide
```
你还需要安装生成服务器和客户端的接口代码相关工具-如果你还没有安装的话，查看下面的设置指南[ Python快速开始指南](/docs/installation/python.html)。

## 定义服务

你的第一步(可以从[概览](/docs/index.html)中得知)是使用 [protocol buffers] (https://developers.google.com/protocol-buffers/docs/overview)去定义 gRPC *service* 和方法 *request* 以及 *response* 的类型。你可以在[`examples/protos/route_guide.proto`](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/protos/route_guide.proto)看到完整的 .proto 文件。

要定义一个服务，你必须在你的 .proto 文件中指定 `service`：

```protobuf
service RouteGuide {
   // (Method definitions not shown)
}
```

然后在你的服务中定义 `rpc` 方法，指定请求的和响应类型。gRPC 允许你定义4种类型的 service 方法，在 `RouteGuide` 服务中都有使用：

- 一个 *简单 RPC* ， 客户端使用存根发送请求到服务器并等待响应返回，就像平常的函数调用一样。

```protobuf
   // Obtains the feature at a given position.
   rpc GetFeature(Point) returns (Feature) {}
```

- 一个 *应答流式 RPC* ， 客户端发送请求到服务器，拿到一个流去读取返回的消息序列。 客户端读取返回的流，直到里面没有任何消息。从例子中可以看出，通过在 *响应* 类型前插入 `stream` 关键字，可以指定一个服务器端的流方法。

```protobuf
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

- 一个 *请求流式 RPC* ， 客户端写入一个消息序列并将其发送到服务器，同样也是使用流。一旦客户端完成写入消息，它等待服务器完成读取返回它的响应。通过在 *请求* 类型前指定 `stream` 关键字来指定一个客户端的流方法。

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

你的 .proto 文件也包含了所有请求的 protocol buffer 消息类型定义以及在服务方法中使用的响应类型-比如，下面的`Point`消息类型：

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

## 生成客户端和服务器端代码

接下来你需要从 .proto 的服务定义中生成 gRPC 客户端和服务器端的接口。你可以通过 protocol buffer 的编译器 `protoc` 以及一个特殊的 gRPC Python 插件来完成。确保你已经安装了 protoc 并且按照 gRPC Python 插件[installation instructions](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/INSTALL)操作。


安装了 `protoc` 和 gRPC Python 插件后，使用下面的命令来生成 Python 代码：

```
$ protoc -I ../../protos --python_out=. --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_python_plugin` ../../protos/route_guide.proto
```
注意我们在例子代码库中已经提供一个版本的生成代码，运行这个命令会重新生成对应的文件而不是创建一个全新的版本。生成的代码文件叫做 `route_guide_pb2.py` 并且包括：

- 定义在 route_guide.proto 中的消息类
- 定义在 route_guide.proto 中的服务的抽象类
   - `BetaRouteGuideServicer`， 定义了 RouteGuide 服务实现的接口rvice
   - `BetaRouteGuideStub`，可以被客户端用来激活 RouteGuide RPC
- 应用使用的函数
   - `beta_create_RouteGuide_server`，根据已有的 `BetaRouteGuideServicer` 对象创建一个 gRPC 服务器
   - `beta_create_RouteGuide_stub`，客户端可以用来创建一个存根对象

## 创建服务器

首先来看看我们如何创建一个 `RouteGuide` 服务器。如果你只对创建 gRPC 客户端感兴趣，你可以跳过这个部分，直接到[创建客户端](#client) (当然你也可能发现它也很有意思)。

创建和运行 `RouteGuide` 服务可以分为两个部分：
- 实现我们服务定义的生成的服务接口：做我们的服务的实际的“工作”的函数。
- 运行一个 gRPC 服务器，监听来自客户端的请求并传输服务的响应。

你可以从[examples/python/route_guide/route_guide_server.py](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/python/route_guide/route_guide_server.py)看到我们的 `RouteGuide` 服务器的例子。

### 实现RouteGuide

`route_guide_server.py` 有一个实现了生成的 `route_guide_pb2.BetaRouteGuideServicer` 接口的 `RouteGuideServicer` 类：

```python
# RouteGuideServicer provides an implementation of the methods of the RouteGuide service.
class RouteGuideServicer(route_guide_pb2.BetaRouteGuideServicer):
```

`RouteGuideServicer` 实现了 `RouteGuide` 所有的服务方法：

#### 简单 RPC

首先让我们看看最简单的类型 `GetFeature`，它从客户端拿到一个 `Point` 对象，然后从返回包含从数据库拿到的feature信息的 `Feature`.

```python
  def GetFeature(self, request, context):
    feature = get_feature(self.db, request)
    if feature is None:
      return route_guide_pb2.Feature(name="", location=request)
    else:
      return feature
```

方法传入了一个 `route_guide_pb2.Point` 的 RPC 请求，以及一个提供了 RPC-specific 信息，如超时限制，的 `ServicerContext`对象。

#### 应答流式 RPC

现在让我们看看下一个方法。`ListFeatures` 是一个应答流 RPC，它会发送多个 `Feature` 给客户端。

```python
  def ListFeatures(self, request, context):
    left = min(request.lo.longitude, request.hi.longitude)
    right = max(request.lo.longitude, request.hi.longitude)
    top = max(request.lo.latitude, request.hi.latitude)
    bottom = min(request.lo.latitude, request.hi.latitude)
    for feature in self.db:
      if (feature.location.longitude >= left and
          feature.location.longitude <= right and
          feature.location.latitude >= bottom and
          feature.location.latitude <= top):
        yield feature
```

这里的请求信息是 `route_guide_pb2.Rectangle`，客户端想从这里找到 `Feature`。该方法会产生0个或者更多的应答而不是单个的应答。

#### 请求流式 RPC

请求流方法 `RecordRoute` 使用了一个请求值的 [迭代器](https://docs.python.org/2/library/stdtypes.html#iterator-types) 并返回了单个的应答值。

```python
  def RecordRoute(self, request_iterator, context):
    point_count = 0
    feature_count = 0
    distance = 0.0
    prev_point = None

    start_time = time.time()
    for point in request_iterator:
      point_count += 1
      if get_feature(self.db, point):
        feature_count += 1
      if prev_point:
        distance += get_distance(prev_point, point)
      prev_point = point

    elapsed_time = time.time() - start_time
    return route_guide_pb2.RouteSummary(point_count=point_count,
                                        feature_count=feature_count,
                                        distance=int(distance),
                                        elapsed_time=int(elapsed_time))
```

#### 双向流式 RPC

最后让我们来看看双向流方法 `RouteChat`。

```python
  def RouteChat(self, request_iterator, context):
    prev_notes = []
    for new_note in request_iterator:
      for prev_note in prev_notes:
        if prev_note.location == new_note.location:
          yield prev_note
      prev_notes.append(new_note)
```

方法的语义是请求流方法和应答流方法的结合。它传入请求值的迭代器并且它本身也是应答值的迭代器。

### 启动服务器

一旦我们实现了所有的 `RouteGuide` 方法，下一步就是启动一个gRPC服务器，这样客户端才可以使用服务：

```python
def serve():
  server = route_guide_pb2.beta_create_RouteGuide_server(RouteGuideServicer())
  server.add_insecure_port('[::]:50051')
  server.start()
```

因为 `start()` 不会阻塞，如果运行时没有其它的代码，你可能需要循环等待。

## 创建客户端

你可以在 [examples/python/route_guide/route_guide_client.py](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/python/route_guide/route_guide_client.py)看到完整的例子代码。

### 创建一个存根

为了能调用服务的方法，我们得先创建一个 *存根*。

我们使用 .proto 中生成的 `route_guide_pb2` 模块的函数`beta_create_RouteGuide_stub`。

```python
channel = implementations.insecure_channel('localhost', 50051)
stub = beta_create_RouteGuide_stub(channel)
```

返回的对象实现了定义在 `BetaRouteGuideStub` 接口中的所有对象。

### 调用服务方法

对于返回单个应答的 RPC 方法（"response-unary" 方法），gRPC Python 同时支持同步（阻塞）和异步（非阻塞）的控制流语义。对于应带流式 RPC 方法，调用会立即返回一个应答值的迭代器。调用迭代器的 `next()` 方法会阻塞，直到从迭代器产生的应答变得可用。

#### 简单 RPC

同步调用简单 RPC `GetFeature` 几乎是和调用一个本地方法一样直观。RPC 调用等待服务器应答，它要么返回应答，要么引起异常：

```python
feature = stub.GetFeature(point, timeout_in_seconds)
```

`GetFeature` 的异步调用很类似，但和在一个线程池里异步调用一个本地方法很像：

```python
feature_future = stub.GetFeature.future(point, timeout_in_seconds)
feature = feature_future.result()
```

#### 应答流 RPC

调用应答流 `ListFeatures` 和使用序列类型类似：

```python
for feature in stub.ListFeatures(rectangle, timeout_in_seconds):
```

#### 请求流 RPC

调用请求流 `RecordRoute` 和给一个本地方法传入序列类似。和前面的简单 RPC 一样，它也会返回单个应答，可以被同步或者异步调用：

```python
route_summary = stub.RecordRoute(point_sequence, timeout_in_seconds)
```

```python
route_summary_future = stub.RecordRoute.future(point_sequence, timeout_in_seconds)
route_summary = route_summary_future.result()
```

#### 双向流 RPC

调用双向流 `RouteChat` 是请求流和应答流语义的结合（这个场景是在服务器端）：

```python
for received_route_note in stub.RouteChat(sent_routes, timeout_in_seconds):
```

## 来试试吧！

运行服务器，它会监听50051端口：

```
$ python route_guide_server.py
```

在另一个终端运行客户端：

```
$ python route_guide_client.py
```
