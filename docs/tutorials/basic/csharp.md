# gRPC 基础： C#

本教程提供了 C# 程序员如何使用 gRPC 的指南。


通过学习教程中例子，你可以学会如何:

- 在一个 .proto 文件内定义服务。
- 用 protocol buffer 编译器生成服务器和客户端代码。
- 使用 gRPC 的 C# API 为你的服务实现一个简单的客户端和服务器。


假设你已经阅读了[概览](/docs/index.html)并且熟悉[protocol buffers](https://developers.google.com/protocol-buffers/docs/overview)。 注意，教程中的例子使用的是 protocol buffers 语言的 proto3 版本，它目前只是 alpha 版:可以在[ proto3 语言指南](https://developers.google.com/protocol-buffers/docs/proto3)和 protocol buffers 的 Github 仓库的[版本注释](https://github.com/google/protobuf/releases)发现更多关于新版本的内容。

这算不上是一个在 C# 中使用 gRPC 的综合指南：以后会有更多的参考文档。

<div id="toc"></div>

## 为什么使用 gRPC?

我们的例子是一个简单的路由映射的应用，它允许客户端获取路由特性的信息，生成路由的总结，以及交互路由信息，如服务器和其他客户端的流量更新。

有了 gRPC， 我们可以一次性的在一个 .proto 文件中定义服务并使用任何支持它的语言去实现客户端和服务器，反过来，它们可以在各种环境中，从Google的服务器到你自己的平板电脑—— gRPC 帮你解决了不同语言及环境间通信的复杂性。使用 protocol buffers 还能获得其他好处，包括高效的序列号，简单的 IDL 以及容易进行的接口更新。

## 例子代码和设置

教程的代码在这里 [grpc/grpc/examples/cpp/route_guide](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/csharp/route_guide). 要下载例子，请通过运行下面的命令去克隆`grpc`代码库:

```
$ git clone https://github.com/grpc/grpc.git
```

本教程的所有文件都在`examples/csharp/route_guide`目录下。
从 Visual Studio (或者 Linux 上的 Monodevelop)打开解决方案 `examples/csharp/route_guide/RouteGuide.sln`。

如果系统是Windows，除了打开解决方案文件之外，你应当不用多做任何事情。所有你需要的依赖都会在都会在构建解决方案的过程中通过 `Grpc` NuGet 包自动恢复。

如果系统是 Linux 或者 Mac OS X，为了生成服务器和客户端接口代码，运行例子，你首先需要安装 protobuf 和 gRPC 的 C# 原生依赖。请查看[如何使用指令](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/src/csharp#how-to-use)的文档。

## 定义服务

我们的第一步(可以从[概览](/docs/index.html)中得知)是使用 [protocol buffers] (https://developers.google.com/protocol-buffers/docs/overview)去定义 gRPC *service* 和方法 *request* 以及 *response* 的类型。你可以在[`examples/protos/route_guide.proto`](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/protos/route_guide.proto)看到完整的 .proto 文件。

要定义一个服务，你必须在你的 .proto 文件中指定 `service`:

```protobuf
service RouteGuide {
   ...
}
```

然后在你的服务中定义 `rpc` 方法，指定请求的和响应类型。gRPC 允许你定义4种类型的 service 方法，这些都在 `RouteGuide` 服务中使用：

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

接下来我们需要从 .proto 的服务定义中生成 gRPC 客户端和服务器端的接口。我们通过 protocol buffer 的编译器 `protoc` 以及一个特殊的 gRPC C# 插件来完成。

如果你想自己运行，请确保你已经安装了 protoc 和 gRPC 的 C# 插件。运行的指令因操作系统而异：

- 对于 Windows 系统来说，`Grpc.Tools` 和 `Google.Protobuf` 的 NuGet 包包含了生成代码所需要的二进制文件。
- 对于 Linux 或者 OS X，请确保你查看了[如何使用指令](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/src/csharp#how-to-use)的文档。

以上都完成后，你就可以生成下面的 C# 代码：

- 我们使用 `Google.Protobuf` NuGet 包的`protoc.exe` 和 `Grpc.Tools` NuGet 包的 `grpc_csharp_plugin.exe` （都在 `tools` 目录下）在 Windows 上生成代码。

一般来说，你需要自己添加 `Grpc.Tools` 包到解决方案中，但在本教程中，它已经帮你完成了。下面的命令应当从`examples/csharp/route_guide` 目录运行：

  ```
  > packages\Google.Protobuf.3.0.0-alpha4\tools\protoc.exe -I../../protos --csharp_out RouteGuide --grpc_out RouteGuide --plugin=protoc-gen-grpc=packages\Grpc.Tools.0.7.0\tools\grpc_csharp_plugin.exe ../../protos/route_guide.proto
  ```

- 在 Linux 或者 OS X 系统，我们可以用 Linuxbrew/Homebrew 安装 `protoc` 和 `grpc_csharp_plugin` 依赖。从 route_guide 目录运行下面的命令：

  ```
  $ protoc -I../../protos --csharp_out RouteGuide --grpc_out RouteGuide --plugin=`which grpc_csharp_plugin` ../../protos/route_guide.proto
  ```

根据操作系统的不同，运行对应的命令去重新在 RouteGuide 目录生成下面的文件：

- `RouteGuide/RouteGuide.cs` 定义了一个命名空间 `Routeguide`
  - 它包含了所有填充，序列化的 protocol buffer 代码以及获取我们的请求和响应的类型。
- `RouteGuide/RouteGuideGrpc.cs`, 提供了存根和服务类
   - 定义 RouteGuide 服务实现时继承的接口 `RouteGuide.IRouteGuide`
   - 用来访问远程 RouteGuide 实例的类 `RouteGuide.RouteGuideClient`

<a name="server"></a>
## 创建服务器

首先来看看我们如何创建一个 `RouteGuide` 服务器。如果你只对创建 gRPC 客户端感兴趣，你可以跳过这个部分，直接到[创建客户端](#client) (当然你也可能发现它也很有意思)。

让 `RouteGuide` 服务工作有两个部分：
- 实现我们服务定义的生成的服务接口：做我们的服务的实际的“工作”。
- 运行一个 gRPC 服务器，监听来自客户端的请求并返回服务的响应。


你可以从[examples/csharp/route_guide/RouteGuideServer/RouteGuideImpl.cs](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/csharp/route_guide/RouteGuideServer/RouteGuideImpl.cs)看到我们的 `RouteGuide` 服务器的实现代码。现在让我们近距离研究它是如何工作的。

### 实现RouteGuide

我们可以看出，服务器有一个实现了生成的 `RouteGuide.IRouteGuide` 接口的 `RouteGuideImpl` 类：

```csharp
// RouteGuideImpl provides an implementation of the RouteGuide service.
public class RouteGuideImpl : RouteGuide.IRouteGuide
```

#### 简单RPC

`RouteGuideImpl` 实现了所有的服务方法。让我们先来看看最简单的类型 `GetFeature`，它从客户端拿到一个 `Point` 然后返回包含从数据库拿到的feature信息的 `Feature` 。

```csharp
public Task<Feature> GetFeature(Point request, Grpc.Core.ServerCallContext context)
{
    return Task.FromResult(CheckFeature(request));
}
```

为了 RPC，方法被传入一个上下文（alpha 版的时候为空），指客户端的 `Point` protocol buffer 请求，返回一个 `Feature` protocol buffer。在方法中，我们用适当的信息创建 `Feature` 然后返回。为了允许异步的实现，方法返回 `Task<Feature>` 而不仅是 `Feature`。你可以随意的同步执行你的计算，并在完成后返回结果，就像我们在例子中做的一样。

#### 服务器端流式 RPC

现在让我们来看看稍微复杂点的 —— 一个流式 RPC。`ListFeatures` 是一个服务器端的流式 RPC，所以我们需要给客户端发回多个 `Feature` protocol buffer。

```csharp
// in RouteGuideImpl
public async Task ListFeatures(Rectangle request,
    Grpc.Core.IServerStreamWriter<Feature> responseStream,
    Grpc.Core.ServerCallContext context)
{
    var responses = features.FindAll( (feature) => feature.Exists() && request.Contains(feature.Location) );
    foreach (var response in responses)
    {
        await responseStream.WriteAsync(response);
    }
}
```

如你所见，这里的请求对象是一个 `Rectangle`，客户端期望从中找到 `Feature`，但是我们需要使用异步方法 `WriteAsync` 写入响应到 `IServerStreamWriter` 异步流而不是一个简单的响应。

#### 客户端流式 RPC

类似的，客户端流方法 `RecordRoute` 使用一个[IAsyncEnumerator](https://github.com/Reactive-Extensions/Rx.NET/blob/master/Ix.NET/Source/System.Interactive.Async/IAsyncEnumerator.cs)，使用异步方法 `MoveNext` 和 `Current` 属性去读取请求的流。

```csharp
public async Task<RouteSummary> RecordRoute(Grpc.Core.IAsyncStreamReader<Point> requestStream,
    Grpc.Core.ServerCallContext context)
{
    int pointCount = 0;
    int featureCount = 0;
    int distance = 0;
    Point previous = null;
    var stopwatch = new Stopwatch();
    stopwatch.Start();

    while (await requestStream.MoveNext())
    {
        var point = requestStream.Current;
        pointCount++;
        if (CheckFeature(point).Exists())
        {
            featureCount++;
        }
        if (previous != null)
        {
            distance += (int) previous.GetDistance(point);
        }
        previous = point;
    }

    stopwatch.Stop();

    return new RouteSummary
    {
        PointCount = pointCount,
        FeatureCount = featureCount,
        Distance = distance,
        ElapsedTime = (int)(stopwatch.ElapsedMilliseconds / 1000)
    };
}
```

#### 双向流式 RPC

最后，让我们来看看双向流 RPC `RouteChat`。

```csharp
public async Task RouteChat(Grpc.Core.IAsyncStreamReader<RouteNote> requestStream,
    Grpc.Core.IServerStreamWriter<RouteNote> responseStream,
    Grpc.Core.ServerCallContext context)
{
    while (await requestStream.MoveNext())
    {
        var note = requestStream.Current;
        List<RouteNote> prevNotes = AddNoteForLocation(note.Location, note);
        foreach (var prevNote in prevNotes)
        {
            await responseStream.WriteAsync(prevNote);
        }
    }
}
```
这里的方法同时接收到 `requestStream` 和 `responseStream` 参数。读取请求和客户端流方法 `RecordRoute` 的方式相同。写入响应和服务器端流方法 `ListFeatures` 的方式相同。

### 启动服务器

一旦我们实现了所有的方法，我们还需要启动一个gRPC服务器，这样客户端才可以使用服务。下面这段代码展示了在我们 `RouteGuide` 服务中实现的过程：

```csharp
var features = RouteGuideUtil.ParseFeatures(RouteGuideUtil.DefaultFeaturesFile);

Server server = new Server
{
    Services = { RouteGuide.BindService(new RouteGuideImpl(features)) },
    Ports = { new ServerPort("localhost", Port, ServerCredentials.Insecure) }
};
server.Start();

Console.WriteLine("RouteGuide server listening on port " + port);
Console.WriteLine("Press any key to stop the server...");
Console.ReadKey();

server.ShutdownAsync().Wait();
```
如你所见，我们通过使用 `Grpc.Core.Server` 去构建和启动服务器。为了做到这点，我们需要：

1. 创建 `Grpc.Core.Server` 的一个实例。
2. 创建我们的服务实现类 `RouteGuideImpl` 的一个实例。
3. 通过在 `Services` 集合中添加服务的定义（我们从生成的 `RouteGuide.BindService` 方法中获得服务定义）注册我们的服务实现。
4. 指定想要接受客户端请求的地址和监听的端口。通过往 `Ports` 集合中添加 `ServerPort` 即可完成。
5. 在服务器实例上调用 `Start` 为我们的服务启动一个 RPC 服务器。


<a name="client"></a>
## 创建客户端

在这一部分，我们将会学习用`RouteGuide`创建一个 C# 客户端。可以在[examples/csharp/route_guide/RouteGuideClient/Program.cs](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/csharp/route_guide/RouteGuideClient/Program.cs)查看完整的客户端代码。

### 创建一个存根

为了能调用服务的方法，我们得先创建一个 *存根*。

首先需要为我们的存根创建一个可以连接到 gRPC 服务器的 gRPC channel。然后我们使用 .proto 生成的`RouteGuide`类的`RouteGuide.NewClient`方法。

```csharp
Channel channel = new Channel("127.0.0.1:50052", ChannelCredentials.Insecure)
var client = RouteGuide.NewClient(channel);

// YOUR CODE GOES HERE

channel.ShutdownAsync().Wait();
```

### 调用服务的方法

现在我们来看看如何调用服务的方法。gRPC C# 的每个支持的方法类型都提供了异步的版本。为了方便起见，gPRC C# 也提供了同步方法存根，不过只能用于简单的（单个请求/单个响应）RPC。

#### 简单 RPC

以同步的方式调用简单 RPC `GetFeature` 几乎是和调用一个本地方法一样直观。

```csharp
Point request = new Point { Latitude = 409146138, Longitude = -746188906 };
Feature feature = client.GetFeature(request);
```
如你所见，我们创建并且填充了一个请求的 protocol buffer 对象（例子中为 `Point`），在客户端对象上调用期望的方法，并传入请求。如果 RPC 成功结束，则返回响应的 protocol buffer（在例子中是`Feature`）。否则抛出 `RpcException` 类型的异常，指出问题的状态码。

或者，如果你在异步的上下文环境中，你可以调用这个方法的异步版本并且使用 `await` 关键字来等待结果:

```csharp
Point request = new Point { Latitude = 409146138, Longitude = -746188906 };
Feature feature = await client.GetFeatureAsync(request);
```

#### 流式 RPC

现在来看看我们的流方法。如果你已经读过[创建服务器](#server)，本节的一些内容看上去很熟悉——流式 RPC 是在客户端和服务器两端以一种类似的方式实现的。和简单的调用不同的地方在于客户端方法返回了调用对象的实例。它提供了使用请求/响应流和（或者）异步的结果，取决于你使用的流类型。

下面就是我们称作是服务器端的流方法 `ListFeatures`，它有`IAsyncEnumerator<Feature>`类型的属性`ReponseStream`：

```csharp
using (var call = client.ListFeatures(request))
{
    while (await call.ResponseStream.MoveNext())
    {
        Feature feature = call.ResponseStream.Current;
        Console.WriteLine("Received " + feature.ToString());
    }
}
```

客户端的流方法 `RecordRoute` 的使用和它很相似，除了我们通过 `WriteAsync` 使用 `RequestStream` 属性挨个写入请求，最后使用 `CompleteAsync` 去通知不再需要发送更多的请求。可以通过 `ResponseAsync` 获取方法的结果。

```csharp
using (var call = client.RecordRoute())
{
    foreach (var point in points)
    {
        await call.RequestStream.WriteAsync(point);
    }
    await call.RequestStream.CompleteAsync();

    RouteSummary summary = await call.ResponseAsync;
}
```

最后，让我们看看双向流式 RPC `RouteChat()`。在这种场景下，我们将请求写入 `RequestStream` 并且从 `ResponseStream` 接受到响应。从例子可以看出，流之间是互相独立的。

```csharp
using (var call = client.RouteChat())
{
    var responseReaderTask = Task.Run(async () =>
    {
        while (await call.ResponseStream.MoveNext())
        {
            var note = call.ResponseStream.Current;
            Console.WriteLine("Received " + note);
        }
    });

    foreach (RouteNote request in requests)
    {
        await call.RequestStream.WriteAsync(request);
    }
    await call.RequestStream.CompleteAsync();
    await responseReaderTask;
}
```

## 来试试吧！

构建客户端和服务器：

- 用 Visual Studio (或者Linux上的Monodevelop) 打开解决方案 `examples/csharp/route_guide/RouteGuide.sln`并选择 **Build**。

- 运行服务器，它会监听50052端口：

  ```
  > cd RouteGuideServer/bin/Debug
  > RouteGuideServer.exe
  ```

- 在另一个终端运行客户端：

  ```
  > cd RouteGuideClient/bin/Debug
  > RouteGuideClient.exe
  ```

你也可以直接从 Visual Studio 里直接运行服务器和客户端。

在Linux 或者 Mac系统中， 使用 `mono RouteGuideServer.exe` 和 `mono RouteGuideClient.exe` 命令去运行服务器和客户端。
