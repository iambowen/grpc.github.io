# gRPC 基础: PHP
本教程提供了 PHP 程序员如何使用 gRPC 的指南。

通过学习教程中例子，你可以学会如何：

- 在一个 .proto 文件内定义服务。
- 用 protocol buffer 编译器生成服务器和客户端代码。
- 使用 gRPC 的 PHP API 为你的服务实现一个简单的客户端和服务器。

假设你已经熟悉[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview)。 注意，教程中的例子使用的是 protocol buffers 语言的 proto2 版本。

同时注意目前你只能用 PHP 创建 gRPC 服务的客户端——你可以从我们的其他教程中，如[Node.js](/docs/tutorials/basic/node.html)，找到如何创建 gRPC 服务器的例子。

这算不上是一个在 PHP 中使用 gRPC 的综合指南：以后会有更多的参考文档。

## 为什么使用 gRPC?

有了 gRPC， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端
和服务器，反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑—— gRPC 帮你解决了
不同语言及环境间通信的复杂性。使用 protocol buffers 还能获得其他好处，包括高效的序
列号，简单的 IDL 以及容易进行接口更新。

## 例子的代码和设置

教程的代码在这里 [grpc/grpc/examples/php/route_guide](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/php/route_guide)。 要下载例子，通过运行下面的命令去克隆`grpc`代码库：

```
$ git clone https://github.com/grpc/grpc.git
```

然后改变当前的目录到 `examples/php/route_guide`:

```
$ cd examples/php/route_guide
```

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互
路由信息，如服务器和其他客户端的流量更新。

你还需要安装生成客户端的接口代码的相关工具（以及一个用其他语言实现的服务器，出于测试的目的）——如果你还没有安装的话，请查看下面的设置指南[这些设置指南](https://github.com/grpc/homebrew-grpc)。

## 来试试吧！

为了使用例子应用，我们需要本地运行一个 gRPC 的服务器。让我们来编译运行，比如这个代码库中的 Node.js 服务器：

```
$ cd ../../node
$ npm install
$ cd route_guide
$ nodejs ./route_guide_server.js --db_path=route_guide_db.json
```

在一个不同的命令窗口运行 PHP 客户端：

```
$ ./run_route_guide_client.sh
```

下面的部分会指导你一步步的理解 proto 服务如何定义，如何从中生成一个客户端类库，以及如何使用类库创建一个应用。

## 定义服务

首先来看看我们使用的服务是如何定义的。gRPC 的 *service* 和它的方法 *request* 以及 *response* 类型使用了[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview)。你可以在[`examples/protos/route_guide.proto`](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/protos/route_guide.proto)看到完整的 .proto 文件。

要定义一个服务，你必须在你的 .proto 文件中指定 `service`：

```protobuf
service RouteGuide {
   ...
}
```
然后在你的服务中定义 `rpc` 方法，指定请求的和响应类型。gRPC 允许你定义4种类型的 service 方法，在 `RouteGuide` 服务中都有使用：

- 一个 *简单 RPC* ， 客户端使用存根发送请求到服务器并等待响应返回，就像平常的远程过程调用调用一样。

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

## 生成客户端代码

可以用[`protoc-gen-php`](https://github.com/datto/protobuf-php)工具从 proto 文件中生成 PHP 客户端存根实现。要安装这个工具，运行：

```sh
$ cd examples/php
$ php composer.phar install
$ cd vendor/datto/protobuf-php
$ gem install rake ronn
$ rake pear:package version=1.0
$ sudo pear install Protobuf-1.0.tgz
```

从 .php 文件 中生成客户端存根实现：

```sh
$ cd php/route_guide
$ protoc-gen-php -i . -o . ./route_guide.proto
```
`php/route_guide` 目录下会生成一个 `route_guide.php` 文件。你不需要修改这个文件。

要加载生成的客户端存根文件，只需要在你的 PHP 应用中 `require` 它：

```php
require dirname(__FILE__) . '/route_guide.php';
```

文件包括：
- 所有用于填充，序列化和获取我们请求和响应消息类型的 protocol buffer 代码
- 一个名为 `examples\RouteGuideClient` 的类，可以让客户端调用定义在 `RouteGuide` 服务中的方法。

## 创建客户端

在这个部分，我们会使用 `RouteGuide` 服务去创建一个 PHP 客户端。在[examples/php/route_guide/route_guide_client.php](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/php/route_guide/route_guide_client.php)可以看到我们完整的客户端例子代码。

### 构造一个客户端对象

要调用一个服务方法，我们首先需要创建一个客户端对象，生成的 `RouteGuideClient` 类的一个实例。该类的构造函数接受一个我们想连接的服务器地址和端口：

```php
$client = new examples\RouteGuideClient('localhost:50051', []);
```

### 调用服务方法

现在让我们来看看如何调用服务方法。

#### 简单 RPC

调用简单 RPC `GetFeature` 几乎是和调用本地的异步方法一样直观。

```php
  $point = new examples\Point();
  $point->setLatitude(409146138);
  $point->setLongitude(-746188906);
  list($feature, $status) = $client->GetFeature($point)->wait();
```

如你所见，我们创建并且填充了一个请求对象，如一个 `examples\Point`。然后，我们调用了存根上的方法，传入请求对象。如果没有错误，那么我们就可以从服务器从应答对象，如一个 `examples\Feature`，中读取应答信息。

```php
  print sprintf("Found %s \n  at %f, %f\n", $feature->getName(),
                $feature->getLocation()->getLatitude() / COORD_FACTOR，
                $feature->getLocation()->getLongitude() / COORD_FACTOR);
```

#### 流式 RPC

现在让我们看看流式方法。下面我们调用了服务器端流方法 `ListFeatures`，它会返回一个地理的 `Feature` 流：

```php
  $lo_point = new examples\Point();
  $hi_point = new examples\Point();

  $lo_point->setLatitude(400000000);
  $lo_point->setLongitude(-750000000);
  $hi_point->setLatitude(420000000);
  $hi_point->setLongitude(-730000000);

  $rectangle = new examples\Rectangle();
  $rectangle->setLo($lo_point);
  $rectangle->setHi($hi_point);

  $call = $client->ListFeatures($rectangle);
  // an iterator over the server streaming responses
  $features = $call->responses();
  foreach ($features as $feature) {
    // process each feature
  } // the loop will end when the server indicates there is no more responses to be sent.
```

`$call->responses()` 方法调用返回一个迭代器。当服务器发送应答时，`foreach` 循环中会返回一个 `$feature` 对象，直到服务器表示没有更多的应答发送。

客户端流方法 `RecordRoute` 的使用很类似，除了我们为每个从客户端写入的每个点调用 `$call->write($point)` ，并拿到一个 `examples\RouteSummary` 返回。

```php
  $call = $client->RecordRoute();

  for ($i = 0; $i < $num_points; $i++) {
    $point = new examples\Point();
    $point->setLatitude($lat);
    $point->setLongitude($long);
    $call->write($point);
  }

  list($route_summary， $status) = $call->wait();
```

最后，让我们看看双向流式 RPC `routeChat()`。在这个场景下，我们给方法传入一个上下文，拿到一个 `BidiStreamingCall` 流对象的返回，我们可以用这个流对象读写消息。

```php
$call = $client->RouteChat();
```

从客户端写入消息：

```php
  foreach ($notes as $n) {
    $route_note = new examples\RouteNote();
    $call->write($route_note);
  }
  $call->writesDone();
```

从服务器读取消息：

```php
  while ($route_note_reply = $call->read()) {
    // process $route_note_reply
  }
```
客户端和服务器获取对方信息的顺序和信息被写入的顺序一致，客户端和服务器都可以以任意顺序读写——流的操作是完全独立的。
