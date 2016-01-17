---
layout: docs
title: gRPC Basics - C++
---

<h1 class="page-header">gRPC基础: C++</h1>

<p class="lead">本教程提供了C++程序员如何使用gRPC的介绍.</p>

通过学习教程中例子，你可以学会如何:

- 在一个.proto定义服务.
- 用protocol buffer编译器生成服务器和客户端代码.
- 使用gRPC的C++ API为你的服务实现一个简单的客户端和服务器.

假设你已经阅读了[概览](/docs/index.html)并且熟悉[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview). 注意，教程中的例子使用的是protocol buffers语言的proto3版本,它目前只是alpha版:可以在[proto3语言指南](https://developers.google.com/protocol-buffers/docs/proto3)和protocol buffers 的Github仓库的[版本注释](https://github.com/google/protobuf/releases)发现更多关于新版本的内容.

这算不上是一个在C++中使用gRPC的综合指南:以后会有更多的参考文档.

<div id="toc"></div>

## 为什么使用gRPC?

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互路由信息，如服务器和其他客户端的流量更新。

有了gRPC, 我们可以一次性的在一个.proto文件中定义服务并使用任何支持它的语言去实现客户端和服务器,反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑-gRPC帮你解决了不同语言间通信的复杂性以及环境的不同.使用protocol buffers还能获得其他好处，包括高效的序列号，简单的IDL以及容易进行接口更新。

## 例子代码和设置

教程的代码在这里 [grpc/grpc/examples/cpp/route_guide](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/cpp/route_guide). 要下载例子，通过运行下面的命令去克隆`grpc`代码库:
```
$ git clone https://github.com/grpc/grpc.git
```

改变当前的目录到`examples/cpp/route_guide`:
```
$ cd examples/cpp/route_guide
```

你还需要安装生成服务器和客户端的接口代码相关工具-如果你还没有安装的话，查看下面的设置指南[ C++快速开始指南](/docs/installation/c.html)。


## 定义服务
我们的第一步(可以从[概览](/docs/index.html)中得知)是使用[protocol buffers] (https://developers.google.com/protocol-buffers/docs/overview)去定义gRPC *service* 和方法 *request* 以及 *response* 的类型。你可以在[`examples/protos/route_guide.proto`](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/protos/route_guide.proto)看到完整的 .proto 文件。

要定义一个服务，你必须在你的.proto文件中指定`service`:

```
service RouteGuide {
   ...
}
```

然后再你的服务中定义`rpc`方法，指定请求的和响应类型。gRPC允许你定义4种类型的service方法，在`RouteGuide`服务中都有使用：

- 一个 *simple RPC* ， 客户端使用桩发送请求到服务器并等待响应返回，就像平常的函数调用一样。

```
   // Obtains the feature at a given position.
   rpc GetFeature(Point) returns (Feature) {}
```

- 一个 *server-side streaming RPC* ， 客户端发送请求到服务器，拿到一个流去读取返回的消息序列。 客户端读取返回的流，直到里面没有任何消息。从例子中可以看出，通过在 *response* 类型前插入`stream`关键字，可以指定一个服务器端的流方法。

```
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

- 一个 *client-side streaming RPC* ， 客户端写入一个消息序列并将其发送到服务器，同样也是使用流。一旦客户端完成写入消息，它等待服务器完成读取返回它的响应。通过在 *request* 类型前指定 `stream`关键字来指定一个客户端的流方法。

```
  // Accepts a stream of Points on a route being traversed, returning a
  // RouteSummary when traversal is completed.
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

- 一个 *bidirectional streaming RPC* 是双方使用读写流去发送一个消息序列。两个流独立操作，因此客户端和服务器可以以任意喜欢的顺序读写：比如， The two streams operate independently, so clients and servers can read and write in whatever order they like: for example, the server could wait to receive all the client messages before writing its responses, or it could alternately read a message then write a message, or some other combination of reads and writes. The order of messages in each stream is preserved. You specify this type of method by placing the `stream` keyword before both the request and the response.

```
  // Accepts a stream of RouteNotes sent while a route is being traversed,
  // while receiving other RouteNotes (e.g. from other users).
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

Our .proto file also contains protocol buffer message type definitions for all the request and response types used in our service methods - for example, here's the `Point` message type:

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

Next we need to generate the gRPC client and server interfaces from our .proto service definition. We do this using the protocol buffer compiler `protoc` with a special gRPC C++ plugin.

For simplicity, we've provided a [makefile](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/cpp/route_guide/Makefile) that runs `protoc` for you with the appropriate plugin, input, and output (if you want to run this yourself, make sure you've installed protoc and followed the gRPC code [installation instructions](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/INSTALL) first):

```
$ make route_guide.grpc.pb.cc route_guide.pb.cc
```

实际上运行的是:

```
$ protoc -I ../../protos --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` ../../protos/route_guide.proto
$ protoc -I ../../protos --cpp_out=. ../../protos/route_guide.proto
```

Running this command generates the following files in your current directory:
- `route_guide.pb.h`, the header which declares your generated message classes
- `route_guide.pb.cc`, which contains the implementation of your message classes
- `route_guide.grpc.pb.h`, the header which declares your generated service classes
- `route_guide.grpc.pb.cc`, which contains the implementation of your service classes

These contain:
- All the protocol buffer code to populate, serialize, and retrieve our request and response message types
- A class called `RouteGuide` that contains
   - a remote interface type (or *stub*) for clients to call with the methods defined in the `RouteGuide` service.
   - two abstract interfaces for servers to implement, also with the methods defined in the `RouteGuide` service.


<a name="server"></a>
## Creating the server

First let's look at how we create a `RouteGuide` server. If you're only interested in creating gRPC clients, you can skip this section and go straight to [Creating the client](#client) (though you might find it interesting anyway!).

There are two parts to making our `RouteGuide` service do its job:
- Implementing the service interface generated from our service definition: doing the actual "work" of our service.
- Running a gRPC server to listen for requests from clients and return the service responses.

You can find our example `RouteGuide` server in [examples/cpp/route_guide/route_guide_server.cc](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/cpp/route_guide/route_guide_server.cc). Let's take a closer look at how it works.

### Implementing RouteGuide

As you can see, our server has a `RouteGuideImpl` class that implements the generated `RouteGuide::Service` interface:

```cpp
class RouteGuideImpl final : public RouteGuide::Service {
...
}
```
In this case we're implementing the *synchronous* version of `RouteGuide`, which provides our default gRPC server behaviour. It's also possible to implement an asynchronous interface, `RouteGuide::AsyncService`, which allows you to further customize your server's threading behaviour, though we won't look at this in this tutorial.

`RouteGuideImpl` implements all our service methods. Let's look at the simplest type first, `GetFeature`, which just gets a `Point` from the client and returns the corresponding feature information from its database in a `Feature`.

```cpp
  Status GetFeature(ServerContext* context, const Point* point,
                    Feature* feature) override {
    feature->set_name(GetFeatureName(*point, feature_list_));
    feature->mutable_location()->CopyFrom(*point);
    return Status::OK;
  }
```

The method is passed a context object for the RPC, the client's `Point` protocol buffer request, and a `Feature` protocol buffer to fill in with the response information. In the method we populate the `Feature` with the appropriate information, and then `return` with an `OK` status to tell gRPC that we've finished dealing with the RPC and that the `Feature` can be returned to the client.

Now let's look at something a bit more complicated - a streaming RPC. `ListFeatures` is a server-side streaming RPC, so we need to send back multiple `Feature`s to our client.

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

As you can see, instead of getting simple request and response objects in our method parameters, this time we get a request object (the `Rectangle` in which our client wants to find `Feature`s) and a special `ServerWriter` object. In the method, we populate as many `Feature` objects as we need to return, writing them to the `ServerWriter` using its `Write()` method. Finally, as in our simple RPC, we `return Status::OK` to tell gRPC that we've finished writing responses.

If you look at the client-side streaming method `RecordRoute` you'll see it's quite similar, except this time we get a `ServerReader` instead of a request object and a single response. We use the `ServerReader`s `Read()` method to repeatedly read in our client's requests to a request object (in this case a `Point`) until there are no more messages: the server needs to check the return value of `Read()` after each call. If `true`, the stream is still good and it can continue reading; if `false` the message stream has ended.

```cpp
while (stream->Read(&point)) {
  ...//process client input
}
```
Finally, let's look at our bidirectional streaming RPC `RouteChat()`.

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

This time we get a `ServerReaderWriter` that can be used to read *and* write messages. The syntax for reading and writing here is exactly the same as for our client-streaming and server-streaming methods. Although each side will always get the other's messages in the order they were written, both the client and server can read and write in any order — the streams operate completely independently.

### Starting the server

Once we've implemented all our methods, we also need to start up a gRPC server so that clients can actually use our service. The following snippet shows how we do this for our `RouteGuide` service:

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
As you can see, we build and start our server using a `ServerBuilder`. To do this, we:

1. Create an instance of our service implementation class `RouteGuideImpl`.
2. Create an instance of the factory `ServerBuilder` class.
3. Specify the address and port we want to use to listen for client requests using the builder's `AddListeningPort()` method.
4. Register our service implementation with the builder.
5. Call `BuildAndStart()` on the builder to create and start an RPC server for our service.
5. Call `Wait()` on the server to do a blocking wait until process is killed or `Shutdown()` is called.

<a name="client"></a>
## Creating the client

In this section, we'll look at creating a C++ client for our `RouteGuide` service. You can see our complete example client code in [examples/cpp/route_guide/route_guide_client.cc](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/cpp/route_guide/route_guide_client.cc).

### 创建一个桩

To call service methods, we first need to create a *stub*.

First we need to create a gRPC *channel* for our stub, specifying the server address and port we want to connect to and any special channel arguments - in our case we'll use the default `ChannelArguments` and no SSL:

```cpp
grpc::CreateChannel("localhost:50051", grpc::InsecureCredentials(), ChannelArguments());
```

Now we can use the channel to create our stub using the `NewStub` method provided in the `RouteGuide` class we generated from our .proto.

```cpp
 public:
  RouteGuideClient(std::shared_ptr<ChannelInterface> channel,
                   const std::string& db)
      : stub_(RouteGuide::NewStub(channel)) {
    ...
  }
```

### 调用服务的方法

Now let's look at how we call our service methods. Note that in this tutorial we're calling the *blocking/synchronous* versions of each method: this means that the RPC call waits for the server to respond, and will either return a response or raise an exception.

#### 简单RPC

调用简单RPC `GetFeature`几乎是和调用一个本地方法一样直观。

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
如你所见，我们创建并且填充了一个请求的protocol buffer对象（例子中为`Point`），同时为了服务器填写创建了一个响应protocol buffer对象。为了调用我们还创建了一个`ClientContext`对象-你可以随意的设置该对象上的配置的值，比如期限，虽然现在我们会使用缺省的设置。注意，你不能在不同的调用间重复使用这个对象。最后，我们在桩上调用这个方法，将其传给上下文，请求以及响应。如果方法的返回是`OK`，那么我们就可以从服务器从我们的响应对象中读取响应信息。

```cpp
      std::cout << "Found feature called " << feature->name()  << " at "
                << feature->location().latitude()/kCoordFactor_ << ", "
                << feature->location().longitude()/kCoordFactor_ << std::endl;
```

#### 流式RPC

现在来看看我们的流方法。如果你已经读过[创建服务器](#server)，本节的一些内容看上去很熟悉-流式RPC是在客户端和服务器两端以一种类似的方式实现的。下面就是我们称作是服务器端的流方法`ListFeatures`，它会返回地理的`Feature`：

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
我们将方法传给上下文并且请求，得到`ClientReader`返回对象，而不是将方法传给上下文，请求和响应。客户端可以使用`ClientReader`去读取服务器的响应。我们使用`ClientReader`的`Read()`反复读取服务器的响应到一个响应protocol buffer对象(在这个例子中是一个`Feature`)，直到没有更多的消息：客户端需要去检查每次调用完`Read()`方法的返回值。如果返回值为`true`，流依然存在并且可以持续读取；如果是`false`，说明消息流已经结束。最后，我们在流上调用`Finish()`方法结束调用并获取我们RPC的状态。

客户端的流方法`RecordRoute`的使用很相似，除了我们将方法传给一个上下文和响应对象，拿到一个`ClientWriter`返回。

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

一旦我们用`Write()`将客户端请求写入到流的动作完成，我们需要在流上调用`WritesDone()`通知gRPC我们已经完成写入，然后调用`Finish()`完成调用同时拿到RPC的状态。如果状态是`OK`，我们最初传给`RecordRoute()`的响应对象会跟着服务器的响应被填充。

最后，让我们看看双向流RPC`RouteChat()`。在这种场景下，我们将上下文传给一个方法，拿到一个可以用来读写消息的`ClientReaderWriter`的返回。

```cpp
    std::shared_ptr<ClientReaderWriter<RouteNote, RouteNote> > stream(
        stub_->RouteChat(&context));
```
这里读写的语法和我们客户端流以及服务器端流方法没有任何区别。虽然每一方都能按照写入时的顺序拿到另一方的消息，客户端和服务器端都可以以任意顺序读写-流操作起来是完全独立的。

## 来试试吧！

构建客户端和服务器:

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
