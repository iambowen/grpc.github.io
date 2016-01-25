---
layout: docs
title: gRPC Basics - Java
---

# gRPC 基础: Java

本教程提供了 Java 程序员如何使用 gRPC 的指南。

通过学习教程中例子，你可以学会如何：

- 在一个 .proto 文件内定义服务.
- 用 protocol buffer 编译器生成服务器和客户端代码.
- 使用 gRPC 的 Java API 为你的服务实现一个简单的客户端和服务器.

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

教程的代码在这里 [grpc/grpc-java/examples/src/main/java/io/grpc/examples](https://github.com/grpc/grpc-java/tree/master/examples/src/main/java/io/grpc/examples)。 要下载例子，通过运行下面的命令去克隆`grpc-java`代码库：


```
$ git clone https://github.com/grpc/grpc-java.git
```

然后改变当前的目录到 `grpc-java/examples`:

```
$ cd grpc-java/examples
```

你还需要安装生成服务器和客户端的接口代码相关工具-如果你还没有安装的话，请查看下面的设置指南[ Java快速开始指南](/docs/installation/java.html)。

## 定义服务

我们的第一步(可以从[概览](/docs/index.html)中得知)是使用 [protocol buffers] (https://developers.google.com/protocol-buffers/docs/overview)去定义 gRPC *service* 和方法 *request* 以及 *response* 的类型。你可以在[`grpc-java/examples/src/main/proto/route_guide.proto`](https://github.com/grpc/grpc-java/blob/master/examples/src/main/proto/route_guide.proto)看到完整的 .proto 文件。

在生成例子中的 Java 代码的时候，在 .proto 文件中我们指定了一个 `java_package` 文件的选项：

```proto
option java_package = "io.grpc.examples";
```

这个指定的包是为我们生成 Java 类使用的。如果在 .proto 文件中没有显示的 `java_package` 参数，
那么就会使用缺省的 proto 包（通过 "package" 关键字指定）。但是，因为 proto 包一般不是以域名
翻转的格式命名，所以它不是好的 Java 包。 如果我们用其它语言通过 .proto 文件生成代码，`java_package` 是不起任何作用的。

要定义一个服务，你必须在你的 .proto 文件中指定 `service`：

```proto
service RouteGuide {
   ...
}
```

然后在我们的服务中定义 `rpc` 方法，指定它们的请求的和响应类型。gRPC允 许你定义4种类型的
service 方法，在 `RouteGuide` 服务中都有使用：

- 一个 *简单 RPC* ， 客户端使用存根发送请求到服务器并等待响应返回，就像平常的函数调用一样。

```proto
   // Obtains the feature at a given position.
   rpc GetFeature(Point) returns (Feature) {}
```

- 一个 *服务器端流式 RPC* ， 客户端发送请求到服务器，拿到一个流去读取返回的消息序列。 客户端读取返回的流，直到里面没有任何消息。从例子中可以看出，通过在 *响应* 类型前插入 `stream` 关键字，可以指定一个服务器端的流方法。

```proto
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

- 一个 *客户端流式 RPC* ， 客户端写入一个消息序列并将其发送到服务器，同样也是使用流。一旦
客户端完成写入消息，它等待服务器完成读取返回它的响应。通过在 *请求* 类型前指定 `stream`
关键字来指定一个客户端的流方法。


```proto
  // Accepts a stream of Points on a route being traversed, returning a
  // RouteSummary when traversal is completed.
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
```
- 一个 *双向流式 RPC* 是双方使用读写流去发送一个消息序列。两个流独立操作，因此客户端和服务器
可以以任意喜欢的顺序读写：比如， 服务器可以在写入响应前等待接收所有的客户端消息，或者可以交替
的读取和写入消息，或者其他读写的组合。 每个流中的消息顺序被预留。你可以通过在请求和响应前加
`stream` 关键字去制定方法的类型。

```proto
  // Accepts a stream of RouteNotes sent while a route is being traversed,
  // while receiving other RouteNotes (e.g. from other users).
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

我们的 .proto 文件也包含了所有请求的 protocol buffer 消息类型定义以及在服务方法中使用的响
应类型——比如，下面的`Point`消息类型：

```proto
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

接下来我们需要从 .proto 的服务定义中生成 gRPC 客户端和服务器端的接口。我们通过 protocol
buffer 的编译器 `protoc` 以及一个特殊的 gRPC Java 插件来完成。为了生成 gRPC 服务，你必须
使用[proto3](https://github.com/google/protobuf/releases)编译器（同时支持 proto2 和
proto3 语法）。

这个例子使用的构建系统也是 Java gRPC 本身构建的一部分——为了简单起见，我们推荐使用为这个例子
提前生成的代码。你可以参考[README](https://github.com/grpc/grpc-java/blob/master/README.md)zh学习如何从你的 .proto 文件中生成代码。

从这里[src/generated/main](https://github.com/grpc/grpc-java/tree/master/examples/src/generated/main)可以看到为了例子预生成的代码。

下面的类都是从我们的服务定义中生成：

- 包含了所有填充，序列化以及获取请求和应答的消息类型的`Feature.java`，`Point.java`，
`Rectangle.java`以及其它类文件。
- `RouteGuideGrpc.java` 文件包含（以及其它一些有用的代码）：
  - `RouteGuide` 服务器要实现的一个接口 `RouteGuideGrpc.RouteGuide`，其中所有的方法都定
  义在`RouteGuide`服务中。
  - 客户端可以用来和`RouteGuide`服务器交互的 *存根* 类。
    异步的存根也实现了 `RouteGuide` 接口。

## 创建服务器

首先来看看我们如何创建一个 `RouteGuide` 服务器。如果你只对创建 gRPC 客户端感兴趣，你可以跳
过这个部分，直接到[创建客户端](#client) (当然你也可能发现它也很有意思)。

让 `RouteGuide` 服务工作有两个部分：
- 实现我们服务定义的生成的服务接口：做我们的服务的实际的“工作”。
- 运行一个 gRPC 服务器，监听来自客户端的请求并返回服务的响应。

你可以从[grpc-java/examples/src/main/java/io/grpc/examples/RouteGuideServer.java]
(https://github.com/grpc/grpc-java/blob/master/examples/src/main/java/io/grpc/examples/routeguide/RouteGuideServer.java)看到我们的 `RouteGuide` 服务器的实现代码。现在让我们近距离研究它是如何工作的。

### 实现RouteGuide

如你所见，我们的服务器有一个实现了生成的 `RouteGuideGrpc.Service` 接口的
`RouteGuideService`类：

```java
private static class RouteGuideService implements RouteGuideGrpc.RouteGuide {
...
}
```
#### 简单 RPC
`routeGuideServer` 实现了我们所有的服务方法。首先让我们看看最简单的类型 `GetFeature`，它
从客户端拿到一个 `Point` 对象，然后从返回包含从数据库拿到的feature信息的 `Feature`.

```java
    @Override
    public void getFeature(Point request, StreamObserver<Feature> responseObserver) {
      responseObserver.onNext(checkFeature(request));
      responseObserver.onCompleted();
    }

...

    private Feature checkFeature(Point location) {
      for (Feature feature : features) {
        if (feature.getLocation().getLatitude() == location.getLatitude()
            && feature.getLocation().getLongitude() == location.getLongitude()) {
          return feature;
        }
      }

      // No feature was found, return an unnamed feature.
      return Feature.newBuilder().setName("").setLocation(location).build();
    }
```

`getFeature()` 接收两个参数：

- `Point`： 请求
- `StreamObserver<Feature>`： 一个应答的观察者，实际上是服务器调用它应答的一个特殊接口。

要将应答返回给客户端，并完成调用：

1. 如在我们的服务定义中指定的那样，我们组织并填充一个 `Feature` 应答对象返回给客户端。在这个
例子中，我们通过一个单独的私有方法`checkFeature()`来实现。
2. 我们使用应答观察者的 `onNext()` 方法返回 `Feature`。
3. 我们使用应答观察者的 `onCompleted()` 方法来指出我们已经完成了和 RPC的交互。

#### 服务器端流式 RPC
现在让我们来看看我们的一种流式 RPC。 `ListFeatures` 是一个服务器端的流式 RPC，所以我们需要将多个 `Feature` 发回给客户端。

```java
private final Collection<Feature> features;

...

    @Override
    public void listFeatures(Rectangle request, StreamObserver<Feature> responseObserver) {
      int left = min(request.getLo().getLongitude(), request.getHi().getLongitude());
      int right = max(request.getLo().getLongitude(), request.getHi().getLongitude());
      int top = max(request.getLo().getLatitude(), request.getHi().getLatitude());
      int bottom = min(request.getLo().getLatitude(), request.getHi().getLatitude());

      for (Feature feature : features) {
        if (!RouteGuideUtil.exists(feature)) {
          continue;
        }

        int lat = feature.getLocation().getLatitude();
        int lon = feature.getLocation().getLongitude();
        if (lon >= left && lon <= right && lat >= bottom && lat <= top) {
          responseObserver.onNext(feature);
        }
      }
      responseObserver.onCompleted();
    }
```

和简单 RPC 类似，这个方法拿到了一个请求对象（客户端期望从 `Rectangle` 找到 `Feature`）和一个应答观察者 `StreamObserver`。

这次我们得到了需要返回给客户端的足够多的 `Feature` 对象（在这个场景下，我们根据他们是否在我们的 `Rectangle` 请求中，从服务的特性集合中选择他们），并且使用 `onNext()` 方法轮流往响应观察者写入。最后，和简单 RPC 的例子一样，我们使用响应观察者的 `onCompleted()` 方法去告诉 gRPC 写入应答已完成。

#### 客户端流 RPC
现在让我们看看稍微复杂点的东西：客户端流方法 `RecordRoute`，我们通过它可以从客户端拿到一个 `Point` 的流，并且返回一个包括它们路径的信息 `RouteSummary`。

```java
 @Override
    public StreamObserver<Point> recordRoute(final StreamObserver<RouteSummary> responseObserver) {
      return new StreamObserver<Point>() {
        int pointCount;
        int featureCount;
        int distance;
        Point previous;
        long startTime = System.nanoTime();

        @Override
        public void onNext(Point point) {
          pointCount++;
          if (RouteGuideUtil.exists(checkFeature(point))) {
            featureCount++;
          }
          // For each point after the first, add the incremental distance from the previous point
          // to the total distance value.
          if (previous != null) {
            distance += calcDistance(previous, point);
          }
          previous = point;
        }

        @Override
        public void onError(Throwable t) {
          logger.log(Level.WARNING, "Encountered error in recordRoute", t);
        }

        @Override
        public void onCompleted() {
          long seconds = NANOSECONDS.toSeconds(System.nanoTime() - startTime);
          responseObserver.onNext(RouteSummary.newBuilder().setPointCount(pointCount)
              .setFeatureCount(featureCount).setDistance(distance)
              .setElapsedTime((int) seconds).build());
          responseObserver.onCompleted();
        }
      };
    }
```
如你所见，这次这个方法没有请求参数。相反的，它拿到了一个 `RouteGuide_RecordRouteServer` 流，服务器可以用它来同时读 *和* 写消息——它可以用自己的 `Recv()` 方法接收客户端消息并且用 `SendAndClose()` 方法返回它的单个响应。

如你所见，我们的方法和前面的方法类型相似，拿到一个 `StreamObserver` 应答观察者参数，但是这次它返回一个 `StreamObserver` 以便客户端写入它的 `Point`。

在这个方法体重，我们返回了一个匿名 `StreamObserver` 实例，其中我们：

- 覆写了 `onNext()` 方法，每次客户端写入一个 `Point` 到消息流时，拿到特性和其它信息。
- 覆写了 `onCompleted()` 方法（在 *客户端* 结束写入消息时调用），用来填充和构建我们的 `RouteSummary`。然后我们用 `RouteSummary` 调用方法自己的的响应观察者的 `onNext()`，之后调用它的 `onCompleted()` 方法，结束服务器端的调用。

#### 双向流式 RPC
最后，让我们看看双向流式 RPC `RouteChat()`。

```java
    @Override
    public StreamObserver<RouteNote> routeChat(final StreamObserver<RouteNote> responseObserver) {
      return new StreamObserver<RouteNote>() {
        @Override
        public void onNext(RouteNote note) {
          List<RouteNote> notes = getOrCreateNotes(note.getLocation());

          // Respond with all previous notes at this location.
          for (RouteNote prevNote : notes.toArray(new RouteNote[0])) {
            responseObserver.onNext(prevNote);
          }

          // Now add the new note to the list
          notes.add(note);
        }

        @Override
        public void onError(Throwable t) {
          logger.log(Level.WARNING, "Encountered error in routeChat", t);
        }

        @Override
        public void onCompleted() {
          responseObserver.onCompleted();
        }
      };
    }
```

和我们的客户端流的例子一样，我们拿到和返回一个 `StreamObserver` 应答观察者，除了这次我们在客户端仍然写入消息到 *它们的* 消息流时通过我们方法的应答观察者返回值。这里读写的语法和客户端流以及服务器流方法一样。虽然每一端都会按照它们写入的顺序拿到另一端的消息，客户端和服务器都可以任意顺序读写——流的操作是互不依赖的。

### 启动服务器

一旦我们实现了所有的方法，我们还需要启动一个gRPC服务器，这样客户端才可以使用服务。下面这段代码展示了在我们`RouteGuide`服务中实现的过程：

```java
  public void start() {
    gRpcServer = NettyServerBuilder.forPort(port)
        .addService(RouteGuideGrpc.bindService(new RouteGuideService(features)))
        .build().start();
    logger.info("Server started, listening on " + port);
    ...
  }
```
如你所见，我们用一个 `NettyServerBuilder` 构建和启动服务器。这个服务器的生成器基于 [Netty](http://netty.io/) 传输框架。

为了做到这个，我们需要：

1. 创建我们服务实现类 `RouteGuideService` 的一个实例并且将其传给生成的 `RouteGuideGrpc` 类的静态方法 `bindService()` 去获得服务定义。
3. 使用生成器的 `forPort()` 方法指定地址以及期望客户端请求监听的端口。
4. 通过传入将 `bindService()` 返回的服务定义，用生成器注册我们的服务实现到生成器的 `addService()` 方法。
5. 调用生成器上的 `build()` 和 `start()` 方法为我们的服务创建和启动一个 RPC 服务器。vice.

## 创建客户端

在这部分，我们将尝试为 `RouteGuide` 服务创建一个 Java 的客户端。你可以从[grpc-java/examples/src/main/java/io/grpc/examples/RouteGuideClient.java](https://github.com/grpc/grpc-java/blob/master/examples/src/main/java/io/grpc/examples/routeguide/RouteGuideClient.java)看到我们完整的客户端例子代码.

### 创建存根

为了调用服务方法，我们需要首先创建一个 *存根*，或者两个存根：

- 一个 *阻塞/同步* 存根：这意味着 RPC 调用等待服务器响应，并且要么返回应答，要么造成异常。
- 一个 *非阻塞/异步* 存根可以向服务器发起非阻塞调用，应答会异步返回。你可以使用异步存根去发起特定类型的流式调用。

我们首先为存根创建一个 gRPC *channel*，指明服务器地址和我们想连接的端口号：

```java
 channel = NettyChannelBuilder.forAddress(host, port)
        .negotiationType(NegotiationType.PLAINTEXT)
        .build();
```

如你所见，我们用一个 `NettyServerBuilder` 构建和启动服务器。这个服务器的生成器基于 [Netty](http://netty.io/) 传输框架。

我们使用 [Netty](http://netty.io/) 传输框架，所以我们用一个 `NettyServerBuilder` 启动服务器。

现在我们可以通过从 .proto 中生成的 `RouteGuideGrpc` 类的 `newStub` 和 `newBlockingStub` 方法，使用频道去创建我们的存根。

```java
    blockingStub = RouteGuideGrpc.newBlockingStub(channel);
    asyncStub = RouteGuideGrpc.newStub(channel);
```

### 调用服务方法

现在让我们看看如何调用服务方法。

#### 简单 RPC

在阻塞存根上调用简单 RPC `GetFeature` 几乎是和调用一个本地方法一样直观。

```java
      Point request = Point.newBuilder().setLatitude(lat).setLongitude(lon).build();
      Feature feature = blockingStub.getFeature(request);
```

我们创建和填充了一个请求 protocol buffer 对象（在这个场景下是 `Point`），在我们的阻塞存根上将其传给 `getFeature()` 方法，拿回一个 `Feature`。

#### 服务器端流式 RPC

Next, let's look at a server-side streaming call to `ListFeatures`, which returns a stream of geographical `Feature`s:

```java
      Rectangle request =
          Rectangle.newBuilder()
              .setLo(Point.newBuilder().setLatitude(lowLat).setLongitude(lowLon).build())
              .setHi(Point.newBuilder().setLatitude(hiLat).setLongitude(hiLon).build()).build();
      Iterator<Feature> features = blockingStub.listFeatures(request);
```

如你所见，这和我们刚看过的简单 RPC 很相似，除了方法返回客户端用来读取所有返回的 `Feature` 的 一个 `Iterator`，而不是单个的 `Feature`。

#### 客户端流式 RPC

现在看看稍微复杂点的东西：我们在客户端流方法 `RecordRoute`发送了一个 `Point` 流给服务器并且拿到一个 `RouteSummary`。为了这个方法，我们需要使用异步存根。如果你已经阅读了
[创建服务器](#server)，一些部分看起来很相近——异步流式 RPC 是在两端通过相似的方式实现的。

```java
  public void recordRoute(List<Feature> features, int numPoints) throws Exception {
    info("*** RecordRoute");
    final SettableFuture<Void> finishFuture = SettableFuture.create();
    StreamObserver<RouteSummary> responseObserver = new StreamObserver<RouteSummary>() {
      @Override
      public void onNext(RouteSummary summary) {
        info("Finished trip with {0} points. Passed {1} features. "
            + "Travelled {2} meters. It took {3} seconds.", summary.getPointCount(),
            summary.getFeatureCount(), summary.getDistance(), summary.getElapsedTime());
      }

      @Override
      public void onError(Throwable t) {
        finishFuture.setException(t);
      }

      @Override
      public void onCompleted() {
        finishFuture.set(null);
      }
    };

    StreamObserver<Point> requestObserver = asyncStub.recordRoute(responseObserver);
    try {
      // Send numPoints points randomly selected from the features list.
      StringBuilder numMsg = new StringBuilder();
      Random rand = new Random();
      for (int i = 0; i < numPoints; ++i) {
        int index = rand.nextInt(features.size());
        Point point = features.get(index).getLocation();
        info("Visiting point {0}, {1}", RouteGuideUtil.getLatitude(point),
            RouteGuideUtil.getLongitude(point));
        requestObserver.onNext(point);
        // Sleep for a bit before sending the next one.
        Thread.sleep(rand.nextInt(1000) + 500);
        if (finishFuture.isDone()) {
          break;
        }
      }
      info(numMsg.toString());
      requestObserver.onCompleted();

      finishFuture.get();
      info("Finished RecordRoute");
    } catch (Exception e) {
      requestObserver.onError(e);
      logger.log(Level.WARNING, "RecordRoute Failed", e);
      throw e;
    }
  }
```

如你所见，为了调用这个方法我们需要创建一个 `StreamObserver`，它为了服务器用它的 `RouteSummary` 应答实现了一个特殊的接口。在 `StreamObserver` 中，我们：

- 覆写了 `onNext()` 方法，在服务器把 `RouteSummary` 写入到消息流时，打印出返回的信息。
- 覆写了 `onCompleted()` 方法（在 *服务器* 完成自己的调用时调用）去设置 `SettableFuture`，这样我们可以检查服务器是不是完成写入。

之后，我们将 `StreamObserver` 传给异步存根的 `recordRoute()` 方法，拿到我们自己的 `StreamObserver` 请求观察者将 `Point` 发给服务器。一旦完成点的写入，我们使用请求观察者的 `onCompleted()` 方法告诉 gRPC 我们已经完成了客户端的写入。一旦我们完成，我们检查 `SettableFuture` 验证服务器已经完成写入。

#### 双向流式 RPC

最后，让我们看看双向流式 RPC `RouteChat()`。

```java
  public void routeChat() throws Exception {
    info("*** RoutChat");
    final SettableFuture<Void> finishFuture = SettableFuture.create();
    StreamObserver<RouteNote> requestObserver =
        asyncStub.routeChat(new StreamObserver<RouteNote>() {
          @Override
          public void onNext(RouteNote note) {
            info("Got message \"{0}\" at {1}, {2}", note.getMessage(), note.getLocation()
                .getLatitude(), note.getLocation().getLongitude());
          }

          @Override
          public void onError(Throwable t) {
            finishFuture.setException(t);
          }

          @Override
          public void onCompleted() {
            finishFuture.set(null);
          }
        });

    try {
      RouteNote[] requests =
          {newNote("First message", 0, 0), newNote("Second message", 0, 1),
              newNote("Third message", 1, 0), newNote("Fourth message", 1, 1)};

      for (RouteNote request : requests) {
        info("Sending message \"{0}\" at {1}, {2}", request.getMessage(), request.getLocation()
            .getLatitude(), request.getLocation().getLongitude());
        requestObserver.onNext(request);
      }
      requestObserver.onCompleted();

      finishFuture.get();
      info("Finished RouteChat");
    } catch (Exception t) {
      requestObserver.onError(t);
      logger.log(Level.WARNING, "RouteChat Failed", t);
      throw t;
    }
  }
```

和我们的客户端流的例子一样，我们拿到和返回一个 `StreamObserver` 应答观察者，除了这次我们在客户端仍然写入消息到 *它们的* 消息流时通过我们方法的应答观察者返回值。这里读写的语法和客户端流以及服务器流方法一样。虽然每一端都会按照它们写入的顺序拿到另一端的消息，客户端和服务器都可以任意顺序读写——流的操作是互不依赖的。

## 来试试吧！

根据example目录下的[README](https://github.com/grpc/grpc-java/blob/master/examples/README.md)的指导去构建和运行客户端及服务器。
