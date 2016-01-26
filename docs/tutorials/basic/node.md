---
layout: docs
title: gRPC Basics - Node.js
---

# gRPC 基础： Node.js

本教程提供了 Node.js 程序员如何使用 gRPC 的指南。

通过学习教程中例子，你可以学会如何：

- 在一个 .proto 文件内定义服务.
- 用 protocol buffer 编译器生成服务器和客户端代码.
- 使用 gRPC 的 Node.js API 为你的服务实现一个简单的客户端和服务器.

假设你已经阅读了[概览](/docs/index.html)并且熟悉[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview)。 注意，教程中的例子使用的是 protocol buffers 语言的 proto3 版本，它目前只是 alpha 版：可以在[ proto3 语言指南](https://developers.google.com/protocol-buffers/docs/proto3)和 protocol buffers 的 Github 仓库的[版本注释](https://github.com/google/protobuf/releases)发现更多关于新版本的内容.

这算不上是一个在 Java 中使用 gRPC 的综合指南：以后会有更多的参考文档.

## 为什么使用 gRPC?

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互
路由信息，如服务器和其他客户端的流量更新。

有了 gRPC， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端
和服务器，反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑- gRPC 帮你解决了
不同语言间通信的复杂性以及环境的不同.使用 protocol buffers 还能获得其他好处，包括高效的序
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

你还需要安装生成服务器和客户端的接口代码相关工具-如果你还没有安装的话，请查看下面的设置指南[ Node.js 快速开始指南](/docs/installation/node.html)。

## 定义服务

我们的第一步(可以从[概览](/docs/index.html)中得知)是使用 [protocol buffers] (https://developers.google.com/protocol-buffers/docs/overview)去定义 gRPC *service* 和方法 *request* 以及 *response* 的类型。你可以在[`grpc-java/examples/src/main/proto/route_guide.proto`](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/protos/route_guide.proto)看到完整的 .proto 文件。

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

## Loading service descriptors from proto files

The Node.js library dynamically generates service descriptors and client stub definitions from `.proto` files loaded at runtime.

To load a `.proto` file, simply `require` the gRPC library, then use its `load()` method:

```js
var grpc = require('grpc');
var protoDescriptor = grpc.load(__dirname + '/route_guide.proto');
// The protoDescriptor object has the full package hierarchy
var example = protoDescriptor.examples;
```

Once you've done this, the stub constructor is in the `examples` namespace (`protoDescriptor.examples.RouteGuide`) and the service descriptor (which is used to create a server) is a property of the stub (`protoDescriptor.examples.RouteGuide.service`);

## Creating the server

First let's look at how we create a `RouteGuide` server. If you're only interested in creating gRPC clients, you can skip this section and go straight to [Creating the client](#client) (though you might find it interesting anyway!).

There are two parts to making our `RouteGuide` service do its job:
- Implementing the service interface generated from our service definition: doing the actual "work" of our service.
- Running a gRPC server to listen for requests from clients and return the service responses.

You can find our example `RouteGuide` server in [examples/node/route_guide/route_guide_server.js](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/node/route_guide/route_guide_server.js). Let's take a closer look at how it works.

### Implementing RouteGuide

As you can see, our server has a `Server` constructor generated from the `RouteGuide.service` descriptor object

```js
var Server = grpc.buildServer([examples.RouteGuide.service]);
```
In this case we're implementing the *asynchronous* version of `RouteGuide`, which provides our default gRPC server behaviour.

The functions in `route_guide_server.js` implement all our service methods. Let's look at the simplest type first, `getFeature`, which just gets a `Point` from the client and returns the corresponding feature information from its database in a `Feature`.

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

The method is passed a call object for the RPC, which has the `Point` parameter as a property, and a callback to which we can pass our returned `Feature`. In the method body we populate a `Feature` corresponding to the given point and pass it to the callback, with a null first parameter to indicate that there is no error.

Now let's look at something a bit more complicated - a streaming RPC. `listFeatures` is a server-side streaming RPC, so we need to send back multiple `Feature`s to our client.

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

As you can see, instead of getting the call object and callback in our method parameters, this time we get a `call` object that implements the `Writable` interface. In the method, we create as many `Feature` objects as we need to return, writing them to the `call` using its `write()` method. Finally, we call `call.end()` to indicate that we have sent all messages.

If you look at the client-side streaming method `RecordRoute` you'll see it's quite similar to the unary call, except this time the `call` parameter implements the `Reader` interface. The `call`'s `'data'` event fires every time there is new data, and the `'end'` event fires when all data has been read. Like the unary case, we respond by calling the callback

```js
call.on('data', function(point) {
  // Process user data
});
call.on('end', function() {
  callback(null, result);
});
```

Finally, let's look at our bidirectional streaming RPC `RouteChat()`.

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

This time we get a `call` implementing `Duplex` that can be used to read *and* write messages. The syntax for reading and writing here is exactly the same as for our client-streaming and server-streaming methods. Although each side will always get the other's messages in the order they were written, both the client and server can read and write in any order — the streams operate completely independently.

### Starting the server

Once we've implemented all our methods, we also need to start up a gRPC server so that clients can actually use our service. The following snippet shows how we do this for our `RouteGuide` service:

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

As you can see, we build and start our server with the following steps:

 1. Create a `Server` constructor from the `RouteGuide` service descriptor.
 2. Implement the service methods.
 3. Create an instance of the server by calling the `Server` constructor with the method implementations.
 4. Specify the address and port we want to use to listen for client requests using the instance's `bind()` method.
 5. Call `listen()` on the instance to start the RPC server.

<a name="client"></a>
## Creating the client

In this section, we'll look at creating a Node.js client for our `RouteGuide` service. You can see our complete example client code in [examples/node/route_guide/route_guide_client.js](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/node/route_guide/route_guide_client.js).

### Creating a stub

To call service methods, we first need to create a *stub*. To do this, we just need to call the RouteGuide stub constructor, specifying the server address and port.

```js
new example.RouteGuide('localhost:50051', grpc.Credentials.createInsecure());
```

### Calling service methods

Now let's look at how we call our service methods. Note that all of these methods are asynchronous: they use either events or callbacks to retrieve results.

#### Simple RPC

Calling the simple RPC `GetFeature` is nearly as straightforward as calling a local asynchronous method.

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

As you can see, we create and populate a request object. Finally, we call the method on the stub, passing it the request and callback. If there is no error, then we can read the response information from the server from our response object.

```js
      console.log('Found feature called "' + feature.name + '" at ' +
          feature.location.latitude/COORD_FACTOR + ', ' +
          feature.location.longitude/COORD_FACTOR);
```

#### Streaming RPCs

Now let's look at our streaming methods. If you've already read [Creating the server](#server) some of this may look very familiar - streaming RPCs are implemented in a similar way on both sides. Here's where we call the server-side streaming method `ListFeatures`, which returns a stream of geographical `Feature`s:

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

Instead of passing the method a request and callback, we pass it a request and get a `Readable` stream object back. The client can use the `Readable`'s `'data'` event to read the server's responses. This event fires with each `Feature` message object until there are no more messages: the `'end'` event indicates that the call is done. Finally, the status event fires when the server sends the status.

The client-side streaming method `RecordRoute` is similar, except there we pass the method a callback and get back a `Writable`.

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

Once we've finished writing our client's requests to the stream using `write()`, we need to call `end()` on the stream to let gRPC know that we've finished writing. If the status is `OK`, the `stats` object will be populated with the server's response.

Finally, let's look at our bidirectional streaming RPC `routeChat()`. In this case, we just pass a context to the method and get back a `Duplex` stream object, which we can use to both write and read messages.

```js
var call = client.routeChat();
```

The syntax for reading and writing here is exactly the same as for our client-streaming and server-streaming methods. Although each side will always get the other's messages in the order they were written, both the client and server can read and write in any order — the streams operate completely independently.

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
