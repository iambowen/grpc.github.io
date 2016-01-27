# gRPC 基础: Ruby

本教程提供了 Python 程序员如何使用 gRPC 的指南。

通过学习教程中例子，你可以学会如何：

- 在一个 .proto 文件内定义服务.
- 用 protocol buffer 编译器生成服务器和客户端代码.
- 使用 gRPC 的 Ruby API 为你的服务实现一个简单的客户端和服务器.

假设你已经阅读了[概览](/docs/index.html)并且熟悉[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview). 注意，教程中的例子使用的是 protocol buffers 语言的 proto3 版本，它目前只是 alpha 版：可以在[ proto3 语言指南](https://developers.google.com/protocol-buffers/docs/proto3)和 protocol buffers 的 Github 仓库的[版本注释](https://github.com/google/protobuf/releases)发现更多关于新版本的内容.

这算不上是一个在 Ruby 中使用 gRPC 的综合指南：以后会有更多的参考文档.

## 为什么使用 gRPC?

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互路由信息，如服务器和其他客户端的流量更新。

有了 gRPC， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端
和服务器，反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑- gRPC 帮你解决了
不同语言间通信的复杂性以及环境的不同.使用 protocol buffers 还能获得其他好处，包括高效的序
列号，简单的 IDL 以及容易进行接口更新。

## 例子代码和设置

教程的代码在这里 [grpc/grpc/examples/python/route_guide](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/python/route_guide)。 要下载例子，通过运行下面的命令去克隆`grpc`代码库：

教程的代码在这里[grpc/grpc/examples/ruby/route_guide](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/ruby/route_guide)。要下载例子，通过运行下面的命令去克隆`grpc`代码库：

```
$ git clone https://github.com/grpc/grpc.git
```

改变当前的目录到 `examples/ruby/route_guide`:

```
$ cd examples/ruby/route_guide
```
你还需要安装生成服务器和客户端的接口代码相关工具-如果你还没有安装的话，查看下面的设置指南[ Ruby快速开始指南](/docs/installation/python.html)。

## 定义服务

我们的第一步(可以从[概览](/docs/index.html)中得知)是使用 [protocol buffers] (https://developers.google.com/protocol-buffers/docs/overview)去定义 gRPC *service* 和方法 *request* 以及 *response* 的类型。你可以在[`examples/protos/route_guide.proto`](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/protos/route_guide.proto)看到完整的 .proto 文件。

要定义一个服务，你必须在你的 .proto 文件中指定 `service`：

```protobuf
service RouteGuide {
   ...
}
```

然后在你的服务中定义 `rpc` 方法，指定请求的和响应类型。gRPC 允许你定义4种类型的 service 方法，在 `RouteGuide` 服务中都有使用：

- 一个 *简单 RPC* ， 客户端使用存根发送请求到服务器并等待响应返回，就像平常的函数调用一样。

```protobuf
   // Obtains the feature at a given position.
   rpc GetFeature(Point) returns (Feature) {}
```

- 一个 *服务器端流式 RPC* ， 客户端发送请求到服务器，拿到一个流去读取返回的消息序列。 客户端读取返回的流，直到里面没有任何消息。从例子中可以看出，通过在 *响应* 类型前插入 `stream` 关键字，可以指定一个服务器端的流方法。

```protobuf
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

- 一个 *客户端流式 RPC* ， 客户端写入一个消息序列并将其发送到服务器，同样也是使用流。一旦客户端完成写入消息，它等待服务器完成读取返回它的响应。通过在 *请求* 类型前指定 `stream` 关键字来指定一个客户端的流方法。

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

我们的 .proto 文件也包含了所有请求的 protocol buffer 消息类型定义以及在服务方法中使用的响应类型-比如，下面的`Point`消息类型：

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

接下来我们需要从 .proto 的服务定义中生成 gRPC 客户端和服务器端的接口。我们通过 protocol buffer 的编译器 `protoc` 以及一个特殊的 gRPC Ruby 插件来完成。

如果你想自己运行，确保你已经安装了 protoc 并且先遵照 gRPC Ruby 插件[installation instructions](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/INSTALL)。

一旦这些完成，就可以用下面的命令来生成 ruby 代码。

```
$ protoc -I ../../protos --ruby_out=lib --grpc_out=lib --plugin=protoc-gen-grpc=`which grpc_ruby_plugin` ../../protos/route_guide.proto
```

运行下面的命令可以在 lib 目录下重新生成下面的文件：

- `lib/route_guide.pb` 定义了一个模块 `Examples::RouteGuide`
  - 包含了所有的填充，序列化和获取我们请求和响应消息类型的 protocol buffer 代码
- `lib/route_guide_services.pb`，继承了 `Examples::RouteGuide` 以及存根和服务类
   - 在定义 RouteGuide 服务实现时将 `Service` 类作为基类entations
   - 用来访问远程 RouteGuide的类 `Stub`

## 创建服务器

首先来看看我们如何创建一个 `RouteGuide` 服务器。如果你只对创建 gRPC 客户端感兴趣，你可以跳过这个部分，直接到[创建客户端](#client) (当然你也可能发现它也很有意思)。

让 `RouteGuide` 服务工作有两个部分：
- 实现我们服务定义的生成的服务接口：做我们的服务的实际的“工作”。
- 运行一个 gRPC 服务器，监听来自客户端的请求并返回服务的响应。


你可以从[examples/ruby/route_guide/route_guide_server.rb](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/ruby/route_guide/route_guide_server.rb)看到 `RouteGuide` 服务器的例子。 现在让我们近距离瞧瞧它是如何工作的。

### Implementing RouteGuide

如你所见，我们的服务器有一个继承生成的 `RouteGuide::Service` 的 `ServerImpl` 类：

```ruby
# ServerImpl provides an implementation of the RouteGuide service.
class ServerImpl < RouteGuide::Service
```

`ServerImpl` 实现了所有的服务方法。首先让我们看看最简单的类型 `GetFeature`，它从客户端拿到一个 `Point` 对象，然后从返回包含从数据库拿到的feature信息的 `Feature`.

```ruby
  def get_feature(point, _call)
    name = @feature_db[{
      'longitude' => point.longitude,
      'latitude' => point.latitude }] || ''
    Feature.new(location: point, name: name)
  end
```

方法为 RPC 传入一个 _call，客户端的 `Point` protocol buffer 请求，并且返回一个 `Feature` protocol buffer。在方法中我们用适当的信息创建了 `Feature`，然后 `return`。

现在看看稍微复杂点的东西 —— 一个流 RPC。`ListFeatures` 是一个服务器端流式 RPC，所以我们需要发回多个 `Feature` 给客户端。

```ruby
# in ServerImpl

  def list_features(rectangle, _call)
    RectangleEnum.new(@feature_db, rectangle).each
  end
```

如你所见，这里的请求对象是一个 `Rectangle`，客户端期望从中找到 `Feature`，但是我们需要返回一个产生应答的[Enumerator](http://ruby-doc.org//core-2.2.0/Enumerator.html)而不是一个简单应答。在方法中，我们使用帮助类 `RectangleEnum` 作为一个 Enumerator 的实现。

类似的，客户端流方法 `record_route` 使用一个[Enumerable](http://ruby-doc.org//core-2.2.0/Enumerable.html)，但是这里是从调用对象获得，我们在先前的例子中略过了这点。`call.each_remote_read` 会一次产生由客户端发送的消息。

```ruby
  call.each_remote_read do |point|
    ...
  end
```
最后，让我们来看看双向流式 RPC `route_chat`。

```ruby
  def route_chat(notes)
    q = EnumeratorQueue.new(self)
    t = Thread.new do
      begin
        notes.each do |n|
		...
      end
	end
    q = EnumeratorQueue.new(self)
  ...
    return q.each_item
  end
```

这里方法接收一个[Enumerable](http://ruby-doc.org//core-2.2.0/Enumerable.html)，但是也会返回一个产生应答的[Enumerator](http://ruby-doc.org//core-2.2.0/Enumerator.html)。实现展示了如何设置它们，而后请求和应答才能并行处理。虽然每一端都会按照它们写入的顺序拿到另一端的消息，客户端和服务器都可以任意顺序读写——流的操作是互不依赖的。

### 启动服务器

一旦我们实现了所有的方法，我们还需要启动一个gRPC服务器，这样客户端才可以使用服务。下面这段代码展示了在我们`RouteGuide`服务中实现的过程：

```ruby
  s = GRPC::RpcServer.new
  s.add_http2_port(port, :this_port_is_insecure)
  logger.info("... running insecurely on #{port}")
  s.handle(ServerImpl.new(feature_db))
  s.run_till_terminated
```
如你所见，我们用 `GRPC::RpcServer` 构建和启动服务器。要做到这点，我们：

1. 用服务的实现类 `ServerImpl` 创建一个实例。
2. 使用生成器的 `add_http2_port` 方法指定地址以及期望客户端请求监听的端口。
3. 用 `GRPC::RpcServer` 注册我们的服务实现。
4. 调用 `GRPC::RpcServer` 的 `run` 去为我们的服务创建和启动 RPC 服务。

## 创建客户端

在这部分，我们将尝试为 `RouteGuide` 服务创建一个 Ruby 的客户端。你可以从[examples/ruby/route_guide/route_guide_client.rb](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/ruby/route_guide/route_guide_client.rb)看到我们完整的客户端例子代码.

### 创建存根

为了调用服务方法，我们需要首先创建一个 *存根*：

我们使用从 .proto 生成的模块 `RouteGuide` 的 `Stub` 类。

```ruby
 stub = RouteGuide::Stub.new('localhost:50051')
```

### 调用服务方法

现在让我们看看如何调用服务方法。注意， gRPC Ruby 只提供了 *阻塞/同步* 版本的方法：这意味着 RPC 调用需要等待服务器响应，并且要么返回应答，要么抛出异常。

#### 简单 RPC

调用简单 RPC `GetFeature` 几乎和调用一个本地方法一样直接。

```ruby
GET_FEATURE_POINTS = [
  Point.new(latitude:  409_146_138, longitude: -746_188_906),
  Point.new(latitude:  0, longitude: 0)
]
..
  GET_FEATURE_POINTS.each do |pt|
    resp = stub.get_feature(pt)
	...
    p "- found '#{resp.name}' at #{pt.inspect}"
  end
```

我们创建和填充了一个请求 protocol buffer 对象（在这个场景下是 `Point`），并且创建了一个应答 protocol buffer 对象让服务器去填充。最后，我们调用存根上的方法，传入上下文，请求以及应答。如果方法返回 `OK`，那么我们就可以从服务器从我们的应答对象中读取应答信息。

#### 流式 RPC

现在让我们看看流方法。如果你已经阅读了[Creating the server](#server)部分，这些可能看上去很相似 —— 流方法在两端的实现很类似。这里我们调用了服务器端的流方法 `list_features`，它会返回 `Features` 的 `Enumerable`。

```ruby
  resps = stub.list_features(LIST_FEATURES_RECT)
  resps.each do |r|
    p "- found '#{r.name}' at #{r.location.inspect}"
  end
```

客户端流方法 `record_route` 也很类似，除了我们给服务器传入一个 `Enumerable`。

```ruby
  ...
  reqs = RandomRoute.new(features, points_on_route)
  resp = stub.record_route(reqs.each, deadline)
  ...
```

最后，让我们看看双向流 RPC `route_chat`。在这个场景下，我们传入一个 `Enumerable`，拿到一个 `Enumerable` 返回。

```ruby
  resps = stub.route_chat(ROUTE_CHAT_NOTES)
  resps.each { |r| p "received #{r.inspect}" }
```

虽然这个例子展现还不够好，每个 enumerable 之间都是互不依赖的 —— 客户端和服务器都可以以任意顺序读写 —— 流的操作是独立的。

## 来试试吧！

构建客户端和服务器：

```
$ # from examples/ruby
$ gem install bundler && bundle install
```
运行服务器，它会监听50051端口：

```
$ # from examples/ruby
$ bundle exec route_guide/route_guide_server.rb ../node/route_guide/route_guide_db.json &
```
在另一个终端运行客户端：

```
$ # from examples/ruby
$ bundle exec route_guide/route_guide_client.rb ../node/route_guide/route_guide_db.json &
```
