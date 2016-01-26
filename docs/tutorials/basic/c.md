---
layout: docs
title: gRPC Basics - C++
---

<h1 class="page-header">gRPC基础：C++</h1>

<p class="lead">本教程提供了C++程序员如何使用gRPC的指南.</p>

通过学习教程中例子，你可以学会如何：

- 在一个 .proto 文件内定义服务.
- 用 protocol buffer 编译器生成服务器和客户端代码.
- 使用 gRPC 的 C++ API 为你的服务实现一个简单的客户端和服务器.

假设你已经阅读了[概览](/docs/index.html)并且熟悉[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview). 注意，教程中的例子使用的是 protocol buffers 语言的 proto3 版本，它目前只是 alpha 版：可以在[ proto3 语言指南](https://developers.google.com/protocol-buffers/docs/proto3)和 protocol buffers 的 Github 仓库的[版本注释](https://github.com/google/protobuf/releases)发现更多关于新版本的内容.

这算不上是一个在 C++ 中使用 gRPC 的综合指南：以后会有更多的参考文档.

<div id="toc"></div>

## 为什么使用 gRPC?

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互路由信息，如服务器和其他客户端的流量更新。

有了 gRPC， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端和服务器，反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑- gRPC 帮你解决了不同语言间通信的复杂性以及环境的不同.使用 protocol buffers 还能获得其他好处，包括高效的序列号，简单的 IDL 以及容易进行接口更新。

## 例子代码和设置

教程的代码在这里 [grpc/grpc/examples/cpp/route_guide](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/cpp/route_guide). 要下载例子，通过运行下面的命令去克隆`grpc`代码库：

```
$ git clone https://github.com/grpc/grpc.git
```

改变当前的目录到`examples/cpp/route_guide`：
```
$ cd examples/cpp/route_guide
```

你还需要安装生成服务器和客户端的接口代码相关工具-如果你还没有安装的话，查看下面的设置指南[ C++快速开始指南](/docs/installation/c.html)。


## 定义服务
我们的第一步(可以从[概览](/docs/index.html)中得知)是使用 [protocol buffers] (https://developers.google.com/protocol-buffers/docs/overview)去定义 gRPC *service* 和方法 *request* 以及 *response* 的类型。你可以在[`examples/protos/route_guide.proto`](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/protos/route_guide.proto)看到完整的 .proto 文件。

要定义一个服务，你必须在你的 .proto 文件中指定 `service`：

```
service RouteGuide {
   ...
}
```

然后在你的服务中定义 `rpc` 方法，指定请求的和响应类型。gRPC允 许你定义4种类型的 service 方法，在 `RouteGuide` 服务中都有使用：

- 一个 *简单 RPC* ， 客户端使用存根发送请求到服务器并等待响应返回，就像平常的函数调用一样。

```
   // Obtains the feature at a given position.
   rpc GetFeature(Point) returns (Feature) {}
```

- 一个 *服务器端流式 RPC* ， 客户端发送请求到服务器，拿到一个流去读取返回的消息序列。 客户端读取返回的流，直到里面没有任何消息。从例子中可以看出，通过在 *响应* 类型前插入 `stream` 关键字，可以指定一个服务器端的流方法。

```
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

- 一个 *客户端流式 RPC* ， 客户端写入一个消息序列并将其发送到服务器，同样也是使用流。一旦客户端完成写入消息，它等待服务器完成读取返回它的响应。通过在 *请求* 类型前指定 `stream` 关键字来指定一个客户端的流方法。

```
  // Accepts a stream of Points on a route being traversed, returning a
  // RouteSummary when traversal is completed.
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

- 一个 *双向流式 RPC* 是双方使用读写流去发送一个消息序列。两个流独立操作，因此客户端和服务器可以以任意喜欢的顺序读写：比如， 服务器可以在写入响应前等待接收所有的客户端消息，或者可以交替的读取和写入消息，或者其他读写的组合。 每个流中的消息顺序被预留。你可以通过在请求和响应前加 `stream` 关键字去制定方法的类型。

```
  // Accepts a stream of RouteNotes sent while a route is being traversed,
  // while receiving other RouteNotes (e.g. from other users).
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

我们的 .proto 文件也包含了所有请求的 protocol buffer 消息类型定义以及在服务方法中使用的响应类型-比如，下面的`Point`消息类型：

```
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

接下来我们需要从 .proto 的服务定义中生成 gRPC 客户端和服务器端的接口。我们通过 protocol buffer 的编译器 `protoc` 以及一个特殊的 gRPC C++ 插件来完成。

简单起见，我们提供一个 [makefile](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/cpp/route_guide/Makefile) 帮您用合适的插件，输入，输出去运行 `protoc`(如果你想自己去运行，确保你已经安装了 protoc，并且请遵循下面的 gRPC 代码[安装指南](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/INSTALL))来操作：

```
$ make route_guide.grpc.pb.cc route_guide.pb.cc
```

实际上运行的是：

```
$ protoc -I ../../protos --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` ../../protos/route_guide.proto
$ protoc -I ../../protos --cpp_out=. ../../protos/route_guide.proto
```
运行这个命令可以在当前目录中生成下面的文件：
- `route_guide.pb.h`， 声明生成的消息类的头文件
- `route_guide.pb.cc`， 包含消息类的实现
- `route_guide.grpc.pb.h`， 声明你生成的服务类的头文件
- `route_guide.grpc.pb.cc`， 包含服务类的实现

这些包括：
- 所有的填充，序列化和获取我们请求和响应消息类型的 protocol buffer 代码
- 名为 `RouteGuide` 的类，包含
   - 为了客户端去调用定义在 `RouteGuide` 服务的远程接口类型(或者 *存根* )
   - 让服务器去实现的两个抽象接口，同时包括定义在 `RouteGuide` 中的方法。


## 创建服务器

首先来看看我们如何创建一个 `RouteGuide` 服务器。如果你只对创建 gRPC 客户端感兴趣，你可以跳过这个部分，直接到[创建客户端](#client) (当然你也可能发现它也很有意思)。

让 `RouteGuide` 服务工作有两个部分：
- 实现我们服务定义的生成的服务接口：做我们的服务的实际的“工作”。
- 运行一个 gRPC 服务器，监听来自客户端的请求并返回服务的响应。

你可以从[examples/cpp/route_guide/route_guide_server.cc](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/cpp/route_guide/route_guide_server.cc)看到我们的 `RouteGuide` 服务器的实现代码。现在让我们近距离研究它是如何工作的。

### 实现RouteGuide

我们可以看出，服务器有一个实现了生成的 `RouteGuide::Service` 接口的 `RouteGuideImpl` 类：

```cpp
class RouteGuideImpl final : public RouteGuide::Service {
...
}
```

在这个场景下，我们正在实现 *同步* 版本的`RouteGuide`，它提供了 gRPC 服务器缺省的行为。同时，也有可能去实现一个异步的接口 `RouteGuide::AsyncService`，它允许你进一步定制服务器线程的行为，虽然在本教程中我们并不关注这点。

`RouteGuideImpl` 实现了所有的服务方法。让我们先来看看最简单的类型 `GetFeature`，它从客户端拿到一个 `Point` 然后将对应的特性返回给数据库中的 `Feature`。

```cpp
  Status GetFeature(ServerContext* context, const Point* point,
                    Feature* feature) override {
    feature->set_name(GetFeatureName(*point, feature_list_));
    feature->mutable_location()——>CopyFrom(*point);
    return Status::OK;
  }
```
这个方法为 RPC 传递了一个上下文对象，包含了客户端的 `Point` protocol buffer 请求以及一个填充响应信息的`Feature` protocol buffer。在这个方法中，我们用适当的信息填充 `Feature`，然后返回`OK`的状态，告诉 gRPC 我们已经处理完 RPC，并且 `Feature` 可以返回给客户端。

现在让我们看看更加复杂点的情况——流式RPC。 `ListFeatures` 是一个服务器端的流式 RPC，因此我们需要给客户端返回多个 `Feature`。

```cpp
  Status ListFeatures(ServerContext* context, const Rectangle* rectangle,
                      ServerWriter<Feature>* writer) override {
    auto lo = rectangle->lo();
    auto hi = rectangle->hi();
    long left = std::min(lo.longitude(), hi.longitude());
    long right = std::max(lo.longitude(), hi.longitude());
    long top = std::max(lo.latitude(), hi.latitude());
    long bottom = std::min(lo.latitude(), hi.latitude());
    for (const Feature& f : feature_list_) {
      if (f.location().longitude() >= left &&
          f.location().longitude() <= right &&
          f.location().latitude() >= bottom &&
          f.location().latitude() <= top) {
        writer->Write(f);
      }
    }
    return Status::OK;
  }
```

如你所见，这次我们拿到了一个请求对象(客户端期望在 `Rectangle` 中找到的 `Feature`)以及一个特殊的 `ServerWriter` 对象，而不是在我们的方法参数中获取简单的请求和响应对象。在方法中，根据返回的需要填充足够多的 `Feature` 对象，用 `ServerWriter` 的 `Write()` 方法写入。最后，和我们简单的 RPC 例子相同，我们返回`Status::OK`去告知gRPC我们已经完成了响应的写入。

如果你看过客户端流方法`RecordRoute`，你会发现它很类似，除了这次我们拿到的是一个`ServerReader`而不是请求对象和单一的响应。我们使用 `ServerReader` 的 `Read()` 方法去重复的往请求对象(在这个场景下是一个 `Point`)读取客户端的请求直到没有更多的消息：在每次调用后，服务器需要检查 `Read()` 的返回值。如果返回值为 `true`，流仍然存在，它就可以继续读取；如果返回值为 `false`，则表明消息流已经停止。

```cpp
while (stream->Read(&point)) {
  ...//process client input
}
```
最后，让我们看看双向流RPC`RouteChat()`。

```cpp
  Status RouteChat(ServerContext* context,
                   ServerReaderWriter<RouteNote, RouteNote>* stream) override {
    std::vector<RouteNote> received_notes;
    RouteNote note;
    while (stream->Read(&note)) {
      for (const RouteNote& n : received_notes) {
        if (n.location().latitude() == note.location().latitude() &&
            n.location().longitude() == note.location().longitude()) {
          stream->Write(n);
        }
      }
      received_notes.push_back(note);
    }

    return Status::OK;
  }
```

这次我们得到的 `ServerReaderWriter` 对象可以用来读 *和* 写消息。这里读写的语法和我们客户端流以及服务器流方法是一样的。虽然每一端获取对方信息的顺序和写入的顺序一致，客户端和服务器都可以以任意顺序读写——流的操作是完全独立的。

### 启动服务器

一旦我们实现了所有的方法，我们还需要启动一个gRPC服务器，这样客户端才可以使用服务。下面这段代码展示了在我们`RouteGuide`服务中实现的过程：

```cpp
void RunServer(const std::string& db_path) {
  std::string server_address("0.0.0.0:50051");
  RouteGuideImpl service(db_path);

  ServerBuilder builder;
  builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
  builder.RegisterService(&service);
  std::unique_ptr<Server> server(builder.BuildAndStart());
  std::cout << "Server listening on " << server_address << std::endl;
  server->Wait();
}
```
如你所见，我们通过使用`ServerBuilder`去构建和启动服务器。为了做到这点，我们需要：

1. 创建我们的服务实现类 `RouteGuideImpl` 的一个实例。
2. 创建工厂类 `ServerBuilder` 的一个实例。
3. 在生成器的 `AddListeningPort()` 方法中指定客户端请求时监听的地址和端口。
4. 用生成器注册我们的服务实现。
5. 调用生成器的 `BuildAndStart()` 方法为我们的服务创建和启动一个RPC服务器。
6. 调用服务器的 `Wait()` 方法实现阻塞等待，直到进程被杀死或者 `Shutdown()` 被调用。

## 创建客户端

在这部分，我们将尝试为`RouteGuide`服务创建一个C++的客户端。你可以从[examples/cpp/route_guide/route_guide_client.cc](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/cpp/route_guide/route_guide_client.cc)看到我们完整的客户端例子代码.

### 创建一个存根

为了能调用服务的方法，我们得先创建一个 *存根*。

首先需要为我们的存根创建一个gRPC *channel*，指定我们想连接的服务器地址和端口，以及 channel 相关的参数——在本例中我们使用了缺省的 `ChannelArguments` 并且没有使用SSL：

```cpp
grpc::CreateChannel("localhost:50051", grpc::InsecureCredentials(), ChannelArguments());
```
现在我们可以利用channel，使用从.proto中生成的`RouteGuide`类提供的`NewStub`方法去创建存根。

```cpp
 public:
  RouteGuideClient(std::shared_ptr<ChannelInterface> channel,
                   const std::string& db)
      : stub_(RouteGuide::NewStub(channel)) {
    ...
  }
```

### 调用服务的方法

现在我们来看看如何调用服务的方法。注意，在本教程中调用的方法，都是 *阻塞/同步* 的版本：这意味着 RPC 调用会等待服务器响应，要么返回响应，要么引起一个异常。

#### 简单RPC

调用简单 RPC `GetFeature` 几乎是和调用一个本地方法一样直观。

```cpp
  Point point;
  Feature feature;
  point = MakePoint(409146138, -746188906);
  GetOneFeature(point, &feature);

...

  bool GetOneFeature(const Point& point, Feature* feature) {
    ClientContext context;
    Status status = stub_->GetFeature(&context, point, feature);
    ...
  }
```
如你所见，我们创建并且填充了一个请求的 protocol buffer 对象（例子中为 `Point`），同时为了服务器填写创建了一个响应 protocol buffer 对象。为了调用我们还创建了一个 `ClientContext` 对象——你可以随意的设置该对象上的配置的值，比如期限，虽然现在我们会使用缺省的设置。注意，你不能在不同的调用间重复使用这个对象。最后，我们在存根上调用这个方法，将其传给上下文，请求以及响应。如果方法的返回是`OK`，那么我们就可以从服务器从我们的响应对象中读取响应信息。

```cpp
      std:：cout << "Found feature called " << feature->name()  << " at "
                << feature->location().latitude()/kCoordFactor_ << ", "
                << feature->location().longitude()/kCoordFactor_ << std:：endl;
```

#### 流式RPC

现在来看看我们的流方法。如果你已经读过[创建服务器](#server)，本节的一些内容看上去很熟悉——流式 RPC 是在客户端和服务器两端以一种类似的方式实现的。下面就是我们称作是服务器端的流方法 `ListFeatures`，它会返回地理的 `Feature`：

```cpp
    std::unique_ptr<ClientReader<Feature> > reader(
        stub_->ListFeatures(&context, rect));
    while (reader->Read(&feature)) {
      std::cout << "Found feature called "
                << feature.name() << " at "
                << feature.location().latitude()/kCoordFactor_ << ", "
                << feature.location().longitude()/kCoordFactor_ << std::endl;
    }
    Status status = reader->Finish();
```
我们将上下文传给方法并且请求，得到 `ClientReader` 返回对象，而不是将上下文，请求和响应传给方法。客户端可以使用 `ClientReader` 去读取服务器的响应。我们使用 `ClientReader` 的 `Read()` 反复读取服务器的响应到一个响应 protocol buffer 对象(在这个例子中是一个 `Feature`)，直到没有更多的消息：客户端需要去检查每次调用完 `Read()` 方法的返回值。如果返回值为 `true`，流依然存在并且可以持续读取；如果是 `false`，说明消息流已经结束。最后，我们在流上调用 `Finish()` 方法结束调用并获取我们 RPC 的状态。

客户端的流方法 `RecordRoute` 的使用很相似，除了我们将一个上下文和响应对象传给方法，拿到一个 `ClientWriter` 返回。

```cpp
    std::unique_ptr<ClientWriter<Point> > writer(
        stub_->RecordRoute(&context, &stats));
    for (int i = 0; i < kPoints; i++) {
      const Feature& f = feature_list_[feature_distribution(generator)];
      std::cout << "Visiting point "
                << f.location().latitude()/kCoordFactor_ << ", "
                << f.location().longitude()/kCoordFactor_ << std::endl;
      if (!writer->Write(f.location())) {
        // Broken stream.
        break;
      }
      std::this_thread::sleep_for(std::chrono::milliseconds(
          delay_distribution(generator)));
    }
    writer->WritesDone();
    Status status = writer->Finish();
    if (status.IsOk()) {
      std::cout << "Finished trip with " << stats.point_count() << " points\n"
                << "Passed " << stats.feature_count() << " features\n"
                << "Travelled " << stats.distance() << " meters\n"
                << "It took " << stats.elapsed_time() << " seconds"
                << std::endl;
    } else {
      std::cout << "RecordRoute rpc failed." << std::endl;
    }
```

一旦我们用 `Write()` 将客户端请求写入到流的动作完成，我们需要在流上调用 `WritesDone()` 通知 gRPC 我们已经完成写入，然后调用 `Finish()` 完成调用同时拿到 RPC 的状态。如果状态是 `OK`，我们最初传给 `RecordRoute()` 的响应对象会跟着服务器的响应被填充。

最后，让我们看看双向流式 RPC `RouteChat()`。在这种场景下，我们将上下文传给一个方法，拿到一个可以用来读写消息的`ClientReaderWriter`的返回。

```cpp
    std::shared_ptr<ClientReaderWriter<RouteNote, RouteNote> > stream(
        stub_->RouteChat(&context));
```
这里读写的语法和我们客户端流以及服务器端流方法没有任何区别。虽然每一方都能按照写入时的顺序拿到另一方的消息，客户端和服务器端都可以以任意顺序读写——流操作起来是完全独立的。

## 来试试吧！

构建客户端和服务器：

```
$ make
```
运行服务器，它会监听50051端口：

```
$ ./route_guide_server
```
在另外一个终端运行客户端：

```
$ ./route_guide_client
```
