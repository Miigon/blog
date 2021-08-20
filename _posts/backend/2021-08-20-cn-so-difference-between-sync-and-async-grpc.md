---
title: "[StackOverflow] 同步与异步 gRPC API 的区别"
date: 2021-08-20 23:09:00 +0800
categories: [Backend, Networking, gRPC]
tags: [c++, gRPC]
---

> ------
> 转自 [我的一个 StackOverflow 回答](https://stackoverflow.com/a/68771426/7509248)。因为答案比较长，而且感觉比较有意义，就翻译成了中文发了出来。
> 
> 原问题:
> ------
> 我正在用 gRPC 构建一个要求高吞吐量的服务。但是我现在用 C++ 同步式 gRPC 编写的程序的吞吐量并不高。
> 
> 我已经读过了 gRPC 文档，但是我并没有找到对于同步/异步 API 的区别的清晰解释。我只知道异步 API 可以控制完成队列（completion queue），而对于同步 API 来说是不可视的。
> 
> 我的理解是同步 gRPC 会发送消息到 TCP 层，然后等待收到 "ack"，因此下个消息会被阻塞，而异步 API 会异步地发送消息，而不需要后面的消息等待前面的消息。

**TLDR: 是的，异步 API 发送消息不会造成后面消息等待，而同步 API 在发送/接收数据的时候，会把整个线程阻塞起来。**

gRPC 的异步操作使用 [完成队列（CompletionQueue）](https://grpc.github.io/grpc/cpp/classgrpc_1_1_completion_queue.html)。这个是它的官方教程: https://grpc.io/docs/languages/cpp/async/

完成队列是一个事件队列（event queue）。这里的“事件”可以是数据接收的完成、或时钟 (alarm) 过期等。（简单来说，任何异步操作的完成都是完成队列中的一个事件）

使用 [gRPC 官方异步 API 示例](https://github.com/grpc/grpc/blob/v1.38.0/examples/cpp/helloworld/greeter_async_server.cc)作为例子，重点观察 `CallData` 类和 `HandleRpcs()`：

``` c++
  void HandleRpcs() {
    // 创建一个新的 CallData 实例来服务新客户端
    new CallData(&service_, cq_.get());
    void* tag;  // 唯一地识别一个请求
    bool ok;
    while (true) {
      // 阻塞读取完成队列中的下个事件。事件是通过 tag 唯一地识别的，
      // 在这里，使用 CallData 实例的内存地址作为 tag。
      // 必须检查 Next 的返回值，这个返回值告诉我们是有事件到来
      // 还是 cq_ 正在关闭。
      GPR_ASSERT(cq_->Next(&tag, &ok));
      GPR_ASSERT(ok);
      static_cast<CallData*>(tag)->Proceed();
    }
  }
```

HandleRpcs() 是服务器的主循环。它使用 `cq->Next()`，不断地从完成队列中获取下一个事件，并调用对应的 `Proceed()` 方法（也就是我们用于处理不同状态的请求的方法）。

`CallData` 类的实例代表了一个完整的客户端请求周期：

```c++
  class CallData {
   public:
    // 输入 "service" 实例（在这里代表一个异步服务器）以及用于与 gRPC 运行时进行异步通信的完成队列 "cq"
    CallData(Greeter::AsyncService* service, ServerCompletionQueue* cq)
        : service_(service), cq_(cq), responder_(&ctx_), status_(CREATE) {
      // 马上调用服务逻辑
      Proceed();
    }

    void Proceed() {
      if (status_ == CREATE) {
        // 将这个实例的状态推进到 PROCESS 状态
        status_ = PROCESS;

        // 作为最初 CREATE 状态的一部分，我们*请求*系统开始处理 SayHello // 请求。在这个请求的过程中，"this" 被作为唯一识别请求的 tag
        // （这样一来不同的请求可以用不同的 CallData 实例异步地被处理）
        service_->RequestSayHello(&ctx_, &request_, &responder_, cq_, cq_,
                                  this);
      } else if (status_ == PROCESS) {
        // 在我们处理这个请求之前，创建一个新的 CallData 实例用于处理未来
        // 的新请求。实例会在它的 FINISH 状态流程中释放自己占用的内存
        new CallData(service_, cq_);

        // 实际请求处理流程
        std::string prefix("Hello ");
        reply_.set_message(prefix + request_.name());

        // 处理完成！让 gRPC 运行时知道我们已经完成了，使用这个实例的内存
        // 地址作为事件内唯一识别请求的 tag
        status_ = FINISH;
        responder_.Finish(reply_, Status::OK, this);
      } else {
        GPR_ASSERT(status_ == FINISH);
        // 已经到达 FINISH 状态，释放自身占用内存（CallData）
        delete this;
      }
    }

   private:
    // 与 gRPC 运行时就一个异步服务进行通信的方法
    Greeter::AsyncService* service_;
    // 用于接收异步服务器通知的生产-消费者队列
    ServerCompletionQueue* cq_;
    // RPC 的上下文信息，用于例如压缩、鉴权以及发送元数据给客户等用途。
    ServerContext ctx_;

    // 从客户端接收到了什么
    HelloRequest request_;
    // 向客户端发送回什么
    HelloReply reply_;

    // 用于回复客户端的方法
    ServerAsyncResponseWriter<HelloReply> responder_;

    // 实现一个带有如下状态的小状态机.
    enum CallStatus { CREATE, PROCESS, FINISH };
    CallStatus status_;  // 目前请求状态
  };
```

如你所见，一个 `CallData` 有三个状态： CREATE，PROCESS 和 FINISH。

## 一个完整的请求流程如下：

0. 启动服务时，预分配 *一个* CallData 实例供未来客户端使用。
1. 该 CallData 对象构造时，`service_->RequestSayHello(&ctx_, &request_, &responder_, cq_, cq_, this)` 将被调用，通知 gRPC 开始准备接收 *恰好一个* `SayHello` 请求。  
这时候我们还不知道请求会由谁发出，何时到达，我们只是告诉 gRPC 说我们已经准备好接收了，让 gRPC 在真的接收到时通知我们。  
提供给 `RequestSayHello` 的参数告诉了 gRPC 将上下文信息、请求体以及回复器放在哪里、使用哪个完成队列来通知、以及通知的时候，用于鉴别请求的 tag（在这个例子中，`this` 被作为 tag 使用）。
2. `HandleRpcs()` 运行到 `cq->Next()` 并阻塞。等待下一个事件发生。

__一段时间后....__

3. 客户端发送一个 `SayHello` 请求到服务器，gRPC 开始接收并解码该请求（IO 操作）

__一段时间后....__

4. gRPC 接收请求完成了。它将请求体放入 CallData 对象的 `request_` 成员中（通过我们之前提供的指针），然后创建一个事件（使用`指向 CallData 对象的指针` 作为 tag），并 **将该事件放到完成队列 `cq_` 中**.
5. `HandleRpcs()` 中的循环接收到了该事件（之前阻塞住的 `cq->Next()` 调用此时也返回），并调用 `CallData::Proceed()` 来处理请求。
6. CallData 的 `status_` 属性此时是 `PROCESS`，它做了如下事情：  
	6.1. 创建一个新的 CallData 对象，这样在这个请求后的新请求才能被新对象处理。  
	6.2. 生成当前请求的回复，告诉 gRPC 我们处理完成了，将该回复发送回客户端  
	6.3. gRPC 开始回复的传输 （IO 操作）  
	6.4. `HandleRpcs()` 中的循环迭代一次，再次阻塞在 `cq->Next()`，等待新事件的发生。  

__一段时间后....__

7. gRPC 完成了回复的传输，再次通过在完成队列里放入一个以 CallData 指针为 tag 的事件的方式通知我们。
8. `cq->Next()` 接收到该事件并返回，`CallData::Proceed()` 将 CallData 对象释放（使用 `delete this;`）。`HandleRpcs()` 循环并重新阻塞在 `cq->Next()` 上，等待新事件的发生。

整个过程看似和同步 API 很相似，只是多了对完成队列的控制。然而，通过这种方式，每一个 `一段时间后....` （通常是在等待 IO 操作的完成或等待一个请求出现） `cq->Next()` 不仅可以接收到当前处理的请求的完成事件，还可以接收到其他请求的事件。

**所以假设第一个请求正在等待它的回复数据传输完成时，一个新的请求到达了，`cq->Next()` 可以获得新请求产生的事件，并开始并行处理新请求，而不用等待第一个请求的传输完成**

另一方面，同步 API总是会等待请求的**完全完成**（从开始接收到完成回复）后才会开始另一个请求的接收。这意味着接收请求体以及发送回复数据（IO 操作）的时候，会出现接近 0% 的 CPU 利用率。因为本可以用于请求处理的宝贵的 CPU 时间都浪费在白等上了。

这是我们需要避免的，因为假设一个网络状况较差的客户端（100ms 的往返延迟）对服务器发送请求，那么对于这个客户端发送的每个请求，我们都将需要花费至少 200ms 用于等待 TCP 传输的完成。这会直接将服务器性能降低到约 5 个请求每秒。

假设我们使用异步 API，我们根本就不主动等待任何东西。我们直接告诉 gRPC 一声：“将这个数据发给客户端，但是我不会站在这里等你完成。你搞定后往完成队列里塞一封信就行了，我后面自己去看。”，然后马上继续去处理其他请求。

# 相关信息

你可以看 [同步 API](https://github.com/grpc/grpc/blob/v1.38.0/examples/cpp/helloworld/greeter_server.cc) 和 [异步 API](https://github.com/grpc/grpc/blob/v1.38.0/examples/cpp/helloworld/greeter_async_server.cc) 的服务器各自是怎么编写。

## 最佳性能实践

由 [gRPC C++ 性能小注](https://grpc.github.io/grpc/cpp/md_doc_cpp_perf_notes.html) 提供的性能最佳实践是创建与 CPU 核心数量一样多的线程，并为每一个线程使用一个完成队列（CompletionQueue）。

