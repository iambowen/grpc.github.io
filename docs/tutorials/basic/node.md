---
layout: docs
title: gRPC Basics - Node.js
---

# gRPC 基础： Node.js

本教程提供了 Node.js 程序员如何使用 gRPC 的指南。

通过学习教程中例子，你可以学会如何：

- 在一个 .proto 文件内定义服务。
- 用 protocol buffer 编译器生成服务器和客户端代码。
- 使用 gRPC 的 Node.js API 为你的服务实现一个简单的客户端和服务器。

假设你已经阅读了[概览](/docs/index.html)并且熟悉[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview)。 注意，教程中的例子使用的是 protocol buffers 语言的 proto3 版本，它目前只是 alpha 版：可以在[ proto3 语言指南](https://developers.google.com/protocol-buffers/docs/proto3)和 protocol buffers 的 Github 仓库的[版本注释](https://github.com/google/protobuf/releases)发现更多关于新版本的内容。

这算不上是一个在 Node.js 中使用 gRPC 的综合指南：以后会有更多的参考文档。

## 为什么使用 gRPC?

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互
路由信息，如服务器和其他客户端的流量更新。

有了 gRPC， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端
和服务器，反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑—— gRPC 帮你解决了
不同语言及环境间通信的复杂性。使用 protocol buffers 还能获得其他好处，包括高效的序
列号，简单的 IDL 以及容易进行接口更新。

## 例子的代码和设置

教程的代码在这里 [grpc/grpc/examples/node/route_guide](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/node/route_guide)。 要下载例子，通过运行下面的命令去克隆`grpc`代码库：

```sh
$ git clone https://github.com/grpc/grpc.git
```

然后改变当前的目录到 `examples/node/route_guide`：

```sh
$ cd examples/node/route_guide
```

你还需要安装生成服务器和客户端的接口代码相关工具——如果你还没有安装的话，请查看下面的设置指南[ Node.js 快速开始指南](/docs/installation/node.html)。

## 定义服务

我们的第一步(可以从[概览](/docs/index.html)中得知)是使用 [protocol buffers] (https://developers.google.com/protocol-buffers/docs/overview)去定义 gRPC *service* 和方法 *request* 以及 *response* 的类型。你可以在[`grpc-java/examples/src/main/proto/route_guide.proto`](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/protos/route_guide.proto)看到完整的 .proto 文件。

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

## 从 proto 文件加载服务描述符

Node.js 的类库在运行时加载 `.proto` 中的客户端存根并动态生成服务描述符。

要加载一个 `.proto` 文件，只需要 `require` gRPC 类库，然后使用它的 `load()` 方法：

```js
var grpc = require('grpc');
var protoDescriptor = grpc.load(__dirname + '/route_guide.proto');
// The protoDescriptor object has the full package hierarchy
var example = protoDescriptor.examples;
```

一旦你完成这个，存根构造函数是在 `examples` 命名空间（`protoDescriptor.examples.RouteGuide`）中而服务描述符（用来创建服务器）是存根（`protoDescriptor.examples.RouteGuide.service`）的一个属性。

## 创建服务器

首先来看看我们如何创建一个 `RouteGuide` 服务器。如果你只对创建 gRPC 客户端感兴趣，你可以跳过这个部分，直接到[创建客户端](#client) (当然你也可能发现它也很有意思)。

让 `RouteGuide` 服务工作有两个部分：
- 实现我们服务定义的生成的服务接口：做我们的服务的实际的“工作”。
- 运行一个 gRPC 服务器，监听来自客户端的请求并返回服务的响应。

你可以从[examples/node/route_guide/route_guide_server.js](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/node/route_guide/route_guide_server.js)看到我们的 `RouteGuide` 服务器的实现代码。现在让我们近距离研究它是如何工作的。

### 实现RouteGuide

可以看出，我们的服务器有一个从 `RouteGuide.service` 描述符对象生成的 `Server` 构造函数：

```js
var Server = grpc.buildServer([examples.RouteGuide.service]);
```
在这个场景下，我们实现了 *异步* 版本的 `RouteGuide`，它提供了 gRPC 缺省的行为。

`route_guide_server.js` 中的函数实现了所有的服务方法。首先让我们看看最简单的类型 `getFeature`，它从客户端拿到一个 `Point` 对象，然后返回包含从数据库拿到的feature信息的 `Feature`。

```js
function checkFeature(point) {
  var feature;
  // Check if there is already a feature object for the given point
  for (var i = 0; i < feature_list.length; i++) {
    feature = feature_list[i];
    if (feature.location.latitude === point.latitude &&
        feature.location.longitude === point.longitude) {
      return feature;
    }
  }
  var name = '';
  feature = {
    name: name,
    location: point
  };
  return feature;
}
function getFeature(call, callback) {
  callback(null, checkFeature(call.request));
}
```

该方法传入了 RPC 的把 `Point` 参数作为属性的调用对象，以及一个可以传入我们返回的 `Feature` 的回调函数。在方法中我们根据给出的点去对应的填充 `Feature`，并将其传给回调函数，其中第一个参数为 null，表示没有错误。

现在让我们看看稍微复杂点的东西 —— 流式 RPC。 `listFeatures` 是一个服务器端流式 RPC，所以我们需要发回多个 `Feature` 给客户端。

```js
function listFeatures(call) {
  var lo = call.request.lo;
  var hi = call.request.hi;
  var left = _.min([lo.longitude, hi.longitude]);
  var right = _.max([lo.longitude, hi.longitude]);
  var top = _.max([lo.latitude, hi.latitude]);
  var bottom = _.min([lo.latitude, hi.latitude]);
  // For each feature, check if it is in the given bounding box
  _.each(feature_list, function(feature) {
    if (feature.name === '') {
      return;
    }
    if (feature.location.longitude >= left &&
        feature.location.longitude <= right &&
        feature.location.latitude >= bottom &&
        feature.location.latitude <= top) {
      call.write(feature);
    }
  });
  call.end();
}
```
如你所见，这次我们拿到了一个实现了 `Writable` 接口的 `call` 对象，而不是调用对象和方法参数中的回调函数。
在方法中，我们根据返回的需要填充足够多的 `Feature` 对象，用它的 `write()` 方法写入到 `call`。最后，我们调用 `call.end()` 表示我们已经完成了所有消息的发送。

如果你看过客户端流方法 `RecordRoute`，你会发现它很类似，除了这次 `call` 参数实现了 `Reader` 的接口。 每次有新数据的时候，`call` 的 `'data'` 事件被触发，每次数据读取完成时， `'end'` 事件被触发。和一元的场景一样，我们通过调用回调函数来应答：

```js
call.on('data', function(point) {
  // Process user data
});
call.on('end', function() {
  callback(null, result);
});
```

最后，让我们来看看双向流式 RPC RouteChat()`。

```js
function routeChat(call) {
  call.on('data', function(note) {
    var key = pointKey(note.location);
    /* For each note sent, respond with all previous notes that correspond to
     * the same point */
    if (route_notes.hasOwnProperty(key)) {
      _.each(route_notes[key], function(note) {
        call.write(note);
      });
    } else {
      route_notes[key] = [];
    }
    // Then add the new note to the list
    route_notes[key].push(JSON.parse(JSON.stringify(note)));
  });
  call.on('end', function() {
    call.end();
  });
}
```

这次我们得到的是一个实现了 `Duplex` 的 `call` 对象，可以用来读 *和* 写消息。这里读写的语法和我们客户端流以及服务器流方法是一样的。虽然每一端获取对方信息的顺序和写入的顺序一致，客户端和服务器都可以以任意顺序读写——流的操作是完全独立的。

### 启动服务器

一旦我们实现了所有的方法，我们还需要启动一个gRPC服务器，这样客户端才可以使用服务。下面这段代码展示了在我们`RouteGuide`服务中实现的过程：

```js
function getServer() {
  var server = new grpc.Server();
  server.addProtoService(routeguide.RouteGuide.service, {
    getFeature: getFeature,
    listFeatures: listFeatures,
    recordRoute: recordRoute,
    routeChat: routeChat
  });
  return server;
}
var routeServer = getServer();
routeServer.bind('0.0.0.0:50051', grpc.ServerCredentials.createInsecure());
routeServer.listen();
```

如你所见，我们通过下面的步骤去构建和启动服务器：

1. 通过 `RouteGuide` 服务描述符创建一个 `Server` 构造函数。
2. 实现服务的方法。
3. 通过调用 `Server` 的构造函数以及方法实现去创建一个服务器的实例。
4. 用实例的 `bind()` 方法指定地址以及我们期望客户端请求监听的端口。
5. 调用实例的 `listen()` 方法启动一个RPC服务器。

## 创建客户端

在这部分，我们将尝试为 `RouteGuide` 服务创建一个 Node.js 的客户端。你可以从[examples/node/route_guide/route_guide_client.js](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/node/route_guide/route_guide_client.js)看到我们完整的客户端例子代码。

### 创建一个存根

为了能调用服务的方法，我们得先创建一个 *存根*。要做到这点，我们只需要调用 RouteGuide 的存根构造函数，指定服务器地址和端口。

```js
new example.RouteGuide('localhost:50051', grpc.Credentials.createInsecure());
```

### 调用服务的方法

现在我们来看看如何调用服务的方法。注意这些方法都是异步的：他们使用事件或者回调函数去获得结果。

#### 简单 RPC

调用简单 RPC `GetFeature` 几乎是和调用一个本地的异步方法一样直观。

```js
var point = {latitude: 409146138, longitude: -746188906};
stub.getFeature(point, function(err, feature) {
  if (err) {
    // process error
  } else {
    // process feature
  }
});
```
如你所见，我们创建并且填充了一个请求对象。最后我们调用了存根上的方法，传入请求和回调函数。如果没有错误，就可以从我们的服务器从应答对象读取应答信息。

```js
      console.log('Found feature called "' + feature.name + '" at ' +
          feature.location.latitude/COORD_FACTOR + ', ' +
          feature.location.longitude/COORD_FACTOR);
```

#### 流式 RPC

现在来看看我们的流方法。如果你已经读过[创建服务器](#server)，本节的一些内容看上去很熟悉——流式 RPC 是在客户端和服务器两端以一种类似的方式实现的。下面就是我们称作是服务器端的流方法 `ListFeatures`，它会返回地理的 `Feature`：

```js
var call = client.listFeatures(rectangle);
  call.on('data', function(feature) {
      console.log('Found feature called "' + feature.name + '" at ' +
          feature.location.latitude/COORD_FACTOR + ', ' +
          feature.location.longitude/COORD_FACTOR);
  });
  call.on('end', function() {
    // The server has finished sending
  });
  call.on('status', function(status) {
    // process status
  });
```

我们传给它一个请求并拿回一个 `Readable` 流对象，而不是给方法传入请求和回调函数。客户端可以使用 `Readable` 的 `'data'` 事件去读取服务器的应答。这个事件由每个 `Feature` 消息对象触发，知道没有更多的消息：`'end'` 事件揭示调用已经结束。最后，当服务器发送状态时，触发状态事件。

客户端的流方法 `RecordRoute` 的使用很相似，除了我们将一个回调函数传给方法，拿到一个 `Writable` 返回。

```js
    var call = client.recordRoute(function(error, stats) {
      if (error) {
        callback(error);
      }
      console.log('Finished trip with', stats.point_count, 'points');
      console.log('Passed', stats.feature_count, 'features');
      console.log('Travelled', stats.distance, 'meters');
      console.log('It took', stats.elapsed_time, 'seconds');
    });
    function pointSender(lat, lng) {
      return function(callback) {
        console.log('Visiting point ' + lat/COORD_FACTOR + ', ' +
            lng/COORD_FACTOR);
        call.write({
          latitude: lat,
          longitude: lng
        });
        _.delay(callback, _.random(500, 1500));
      };
    }
    var point_senders = [];
    for (var i = 0; i < num_points; i++) {
      var rand_point = feature_list[_.random(0, feature_list.length - 1)];
      point_senders[i] = pointSender(rand_point.location.latitude,
                                     rand_point.location.longitude);
    }
    async.series(point_senders, function() {
      call.end();
    });
```

一旦我们用 `write()` 将客户端请求写入到流的动作完成，我们需要在流上调用 `end()` 通知 gRPC 我们已经完成写。如果状态是 `OK`，`stats` 对象会跟着服务器的响应被填充。

最后，让我们看看双向流式 RPC `routeChat()`。在这种场景下，我们将上下文传给一个方法，拿到一个可以用来读写消息的 `Duplex` 流对象的返回。

```js
var call = client.routeChat();
```
这里读写的语法和我们客户端流以及服务器端流方法没有任何区别。虽然每一方都能按照写入时的顺序拿到另一方的消息，客户端和服务器端都可以以任意顺序读写——流操作起来是完全独立的。

## 来试试吧！

构建客户端和服务器：

```sh
$ npm install
```
运行服务器，它会监听50051端口：

```sh
$ node ./route_guide_server.js
```
在另一个终端运行客户端：

```sh
$ node ./route_guide_client.js
```
