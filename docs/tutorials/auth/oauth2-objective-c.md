# 在 gRPC 上使用 OAuth2: Objective-C

这个例子展示了如何在 gRPC 上使用 OAuth2 代表用户发起身份验证 API 调用。通过它你还会学到如何
使用 Objective-C gRPC API 去：

- 在 RPC 启动前初始化和配置一个远程调用对象。
- 在一个调用上设置请求的元数据元素，语义上等同于 HTTP 的请求头部。
- 从调用上读取应答的元数据，等同于 HTTP 应答的头和尾。

假设你知道如何使用 Objective-C 的客户端类库去发起 gRPC 调用，如在[gRPC 基础: Objective-C](/docs/tutorials/basic/objective-c.html)和[概览](/docs/index.html)中介绍的那样，以及熟悉 OAuth2 的概念如 _access token_。

## 例子代码和设置

我们教程的例子代码在这里[gprc/examples/objective-c/auth_sample](https://github.com/grpc/grpc/tree/master/examples/objective-c/auth_sample)。要下载这个例子，通过运行下面的命令克隆代码库：

```
$ git clone https://github.com/grpc/grpc.git
$ cd grpc
$ git submodule update --init
```

然后切换目录到 `examples/objective-c/auth_sample`：

```
$ cd examples/objective-c/auth_sample
```

我们的例子是一个有两个view的简单应用。一地个view让用户使用 Google 的[iOS 登陆类库](https://developers.google.com/identity/sign-in/ios/)的 OAuth2 工作流去登陆和登出。（例子中用到了 Google 的类库，是因为我们要调用的测试 gRPC 服务需要 Google 账号身份，但是 gRPC 和 Objective-C 客户端类库都没有连接任何 OAuth2 提供商）。第二个view使用第一个view获得的 access token 向测试服务器发起 gRPC 请求。

注意：OAuth2 类库需要应用注册并且从身份提供者获得一个 ID（在例子应用中是 Google）。应用的 XCode 项目配置使用那个 ID，所以你不应该拷贝这个工程”当做是“自己的应用：这会导致你的应用程序作为 "gRPC-AuthSample" 在同意屏幕被确定，并且不能访问真正的 Google 服务。相反，根据[指南](https://developers.google.com/identity/sign-in/ios/)去配置你自己的 XCode 工程。

在使用其它的 Objective-C 例子时，你应该已经安装了[Cocoapods](https://cocoapods.org/#install)，还有相关的生成客户端类库代码的工具。你也可以按照[这些设置指南](https://github.com/grpc/homebrew-grpc)得到后者。

## 来试试吧！

要试试例子应用呢，首先为我们的 .proto 文件使用 Cocoapods 生成和安装客户端类库：

```
$ pod install
```

（这也许需要编译 OpenSSL，如果你的电脑上没有 Cocoapods 的缓存，这也许要花上 15 分钟左右）。

最后，打开 Cocoapods 创建的 XCode workspace，运行应用。

第一个 view `SelectUserViewController.h/m`，要求你用 Google 账户登录，授于 "gRPC-AuthSample" 应用如下的权限：

- 查看你的邮件地址。
- 查看你基本的档案信息。
- "访问 Zoo 服务的测试范围"。

最后一个权限，对应的范围是 `https://www.googleapis.com/auth/xapi.zoo`，并没有给予任何正式的能力：这只是用来测试。你随时可以登出。

第二个 view `MakeRPCViewController.h/m`，向位于 https://grpc-test.sandbox.google.com 的测试服务器发起 gRPC 请求，包括 access token 。测试服务器只是检验了 token，而后将它所属的用户以及授予访问的范围写入到应答中。（客户端应用已经知道这两个值；这是一种验证所有事情如我们所料的方式）。

下一个部分的指南会一步步指导你如何实行 `MakeRPCViewController` 中的 gRPC 调用。你可以在[MakeRPCViewController.m](https://github.com/grpc/grpc/blob/master/examples/objective-c/auth_sample/MakeRPCViewController.m)看到完整例子的代码。

## 创建 RPC 客户端

另一个基本的教程展示如何通过调用生成的客户端对象中的异步方法来激活一个 RPC。但是，发起身份验证的调用需要你去初始化一个代表 RPC 的对象，在发起网络请求 _前_ 配置好它。首先让我们看看如何创建 RPC 对象。

假设你的 proto 服务定义如下：

```protobuf
option objc_class_prefix = "AUTH";

service TestService {
  rpc UnaryCall(Request) returns (Response);
}
```

为了 `AUTHTestService` 类生成的一个 `unaryCallWithRequest:handler:` 方法，你应该已经很熟悉了：

```objective-c
[client unaryCallWithRequest:request handler:^(AUTHResponse *response, NSError *error) {
  ...
}];
```

此外，一个 `RPCToUnaryCallWithRequest:handler:` 被生成，它会返回一个还没有开始的 RPC 对象：

```objective-c
#import <ProtoRPC/ProtoRPC.h>

ProtoRPC *call =
    [client RPCToUnaryCallWithRequest:request handler:^(AUTHResponse *response, NSError *error) {
      ...
    }];
```

你可以像这样在任何以后的时间开始这个对象代表的 RPC：

```objective-c
[call start];
```
## 设置请求元数据：: 有一个 access token 的身份验证头

现在让我们看看如何配置 RPC 对象上的一些设置。`ProtoRPC` 有个 `requestHeaders` 属性（从 `GRPCCall` 继承）定义如下：

```objective-c
@property(atomic, readonly) id<GRPCRequestHeaders> requestHeaders
```

你可以把 `GRPCRequestHeaders` 协议等同于 `NSMutableDictionary` 类。设置元数据键值的词典的元素意味着这个元数据在调用开始后将会被发送。gRPC 元数据是关于客户端发往服务器调用的信息片（反之亦然）。它们以键值对的形式存在，并且对于 gRPC 本身基本不透明。

方便起见，属性通过空的 `NSMutableDictionary` 初始化，以便请求元数据元素可以像下面一样设置：

```objective-c
call.requestHeaders[@"My-Header"] = @"Value for this header";
call.requestHeaders[@"Another-Header"] = @"Its value";
```

元数据的典型使用是验证细节，像我们例子中一样。如果你已经有了 access token，OAuth2 指定它以下面的格式发送：

```objective-c
call.requestHeaders[@"Authorization"] = [@"Bearer " stringByAppendingString:accessToken];
```

## 拿到应答元数据：验证挑战头

`ProtoRPC` 类也继承了一对属性，`responseHeaders` 和 `responseTrailers`， 类似于请求元数据，我们只是看着而是由服务器向客户端发回。它们的定义如下：

```objective-c
@property(atomic, readonly) NSDictionary *responseHeaders;
@property(atomic, readonly) NSDictionary *responseTrailers;
```

在 OAuth2 中，如果验证出错，服务器会返回一个挑战头。它通过 RPC 的应答头返回。要访问这个，如我们例子中的错误处理的代码，你可以这么写：

```objective-c
call.responseHeaders[@"www-authenticate"]
```

注意，gRPC 的元数据元素可以映射到 HTTP/2 的头（或者尾），应答元数据的键永远是小写的 ASCII 码字符串。
许多应答元数据的使用场景都涉及如何拿到关于一个 RPC 错误的更多细节。简单起见，当 RPC 的处理块传入一个 `NSError` 实例时，应答头和尾词典也可以这种方式访问：

```objective-c
error.userInfo[kGRPCHeadersKey] == call.responseHeaders
error.userInfo[kGRPCTrailersKey] == call.responseTrailers
```
