# 异步基础: C++

本教程介绍如何使用 C++ 的 gRPC 异步/非阻塞 API 去实现简单的服务器和客户端。假设你已经熟悉实现同步 gRPC 代码，如[gRPC 基础: C++](/docs/tutorials/basic/c.html)所描述的。本教程中的例子基本来自我们在[overview](/docs/index.html)中使用的[Greeter 例子](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/cpp/helloworld)。你可以在 [grpc/examples/cpp/helloworld](https://github.com/grpc/grpc/tree/{{ site.data.config.grpc_release_branch }}/examples/cpp/helloworld)找到安装指南。

## 概览

gRPC 的异步操作使用[`CompletionQueue`](http://www.grpc.io/grpc/cpp/classgrpc_1_1_completion_queue.html)。 基本工作流如下：

- 在 RPC 调用上绑定一个 `CompletionQueue`
- 做一些事情如读取或者写入，以唯一的 `voide*` 标签展示
- 调用 `CompletionQueue::Next` 去等待操作结束。如果标签出现，表示对应的操作已经完成。

## 异步客户端

要使用一个异步的客户端调用远程方法，你首先得创建一个频道和存根，如你在[同步客户端](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/cpp/helloworld/greeter_client.cc)中所作的那样。一旦有了存根，你就可以通过下面的方式来做异步调用：

- 初始化 RPC 并为之创建句柄。将 RPC 绑定到一个 `CompletionQueue`。

    ```
    CompletionQueue cq;
    std::unique_ptr<ClientAsyncResponseReader<HelloReply> > rpc(
        stub_->AsyncSayHello(&context, request, &cq));
    ```

- 用一个唯一的标签，寻求回答和最终的状态

    ```
    Status status;
    rpc->Finish(&reply, &status, (void*)1);
    ```

- 等待完成队列返回下一个标签。当标签被传入对应的 `Finish()` 调用时，回答和状态就可以被返回了。

    ```
    void* got_tag;
    bool ok = false;
    cq.Next(&got_tag, &ok);
    if (ok && got_tag == (void*)1) {
      // check reply and status
    }
    ```

你可以在这里[greeter&#95;async&#95;client.cc](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/cpp/helloworld/greeter_async_client.cc)看到完整的客户端例子。

## 异步服务器

服务器实现请求一个带有标签的 RPC 调用，然后等待完成队列返回标签。异步处理 RPC 的基本工作流如下：

- 构建一个服务器导出异步服务

    ```
    helloworld::Greeter::AsyncService service;
    ServerBuilder builder;
    builder.AddListeningPort("0.0.0.0:50051", InsecureServerCredentials());
    builder.RegisterAsyncService(&service);
    auto cq = builder.AddCompletionQueue();
    auto server = builder.BuildAndStart();
    ```

- 请求一个 RPC 提供唯一的标签

    ```
    ServerContext context;
    HelloRequest request;
    ServerAsyncResponseWriter<HelloReply> responder;
    service.RequestSayHello(&context, &request, &responder, &cq, &cq, (void*)1);
    ```

- 等待完成队列返回标签。当取到标签时，上下文，请求和应答器都已经准备就绪。

    ```
    HelloReply reply;
    Status status;
    void* got_tag;
    bool ok = false;
    cq.Next(&got_tag, &ok);
    if (ok && got_tag == (void*)1) {
      // set reply and status
      responder.Finish(reply, status, (void*)2);
    }
    ```

- 等待完成队列返回标签。标签返回时 RPC 结束。

    ```
    void* got_tag;
    bool ok = false;
    cq.Next(&got_tag, &ok);
    if (ok && got_tag == (void*)2) {
      // clean up
    }
    ```

然而，这个基本的工作流没有考虑服务器并发处理多个请求。要解决这个问题，我们的完成异步服务器例子使用了 `CallData` 对象去维护每个 RPC 的状态，并且使用这个对象的地址作为调用的唯一标签。

```
  class CallData {
   public:
    // Take in the "service" instance (in this case representing an asynchronous
    // server) and the completion queue "cq" used for asynchronous communication
    // with the gRPC runtime.
    CallData(Greeter::AsyncService* service, ServerCompletionQueue* cq)
        : service_(service), cq_(cq), responder_(&ctx_), status_(CREATE) {
      // Invoke the serving logic right away.
      Proceed();
    }

    void Proceed() {
      if (status_ == CREATE) {
        // As part of the initial CREATE state, we *request* that the system
        // start processing SayHello requests. In this request, "this" acts are
        // the tag uniquely identifying the request (so that different CallData
        // instances can serve different requests concurrently), in this case
        // the memory address of this CallData instance.
        service_->RequestSayHello(&ctx_, &request_, &responder_, cq_, cq_,
                                  this);
        // Make this instance progress to the PROCESS state.
        status_ = PROCESS;
      } else if (status_ == PROCESS) {
        // Spawn a new CallData instance to serve new clients while we process
        // the one for this CallData. The instance will deallocate itself as
        // part of its FINISH state.
        new CallData(service_, cq_);

        // The actual processing.
        std::string prefix("Hello ");
        reply_.set_message(prefix + request_.name());

        // And we are done! Let the gRPC runtime know we've finished, using the
        // memory address of this instance as the uniquely identifying tag for
        // the event.
        responder_.Finish(reply_, Status::OK, this);
        status_ = FINISH;
      } else {
        GPR_ASSERT(status_ == FINISH);
        // Once in the FINISH state, deallocate ourselves (CallData).
        delete this;
      }
    }
```

简单起见，服务器对于所有的事件只使用了一个完成队列，并且在 `HandleRpcs` 中运行了一个主循环去查询队列：

```
  void HandleRpcs() {
    // Spawn a new CallData instance to serve new clients.
    new CallData(&service_, cq_.get());
    void* tag;  // uniquely identifies a request.
    bool ok;
    while (true) {
      // Block waiting to read the next event from the completion queue. The
      // event is uniquely identified by its tag, which in this case is the
      // memory address of a CallData instance.
      cq_->Next(&tag, &ok);
      GPR_ASSERT(ok);
      static_cast<CallData*>(tag)->Proceed();
    }
  }
```

你可以在[greeter&#95;async&#95;server.cc](https://github.com/grpc/grpc/blob/{{ site.data.config.grpc_release_branch }}/examples/cpp/helloworld/greeter_async_server.cc)看到完整的服务器例子。
