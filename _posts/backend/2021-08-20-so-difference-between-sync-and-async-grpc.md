---
title: "[StackOverflow] Difference between synchronous and asynchorous gRPC API"
date: 2021-08-20 23:09:00 +0800
categories: [Backend, Networking, gRPC]
tags: [c++, gRPC]
---

> ------
> This is from [one of my answers on StackOverflow](https://stackoverflow.com/a/68771426/7509248).
> 
> Original question:
> ------
> I am working on a service based on gRPC, which requires high throughput. But currently my program suffers low throughput when using C++ synchronous gRPC.
> 
> I've read through gRPC documentations, but don't find explicit explanation on the difference between sync/async APIs. Except async has control over completion queue, while it's transparent to sync APIs.
> 
> I want to know whether synchronous gRPC sends messages to TCP layer, and wait for its "ack", thus the next message would be blocked? Meanwhile async APIs would send them asynchronously without latter messages waiting?

**TLDR: Yes, async APIs would send the messages asynchronously without latter messages waiting, while synchronous APIs will block the whole thread while one message is being sent/received.**

gRPC uses [CompletionQueue](https://grpc.github.io/grpc/cpp/classgrpc_1_1_completion_queue.html) for it's asynchronous operations. You can find the official tutorial here: https://grpc.io/docs/languages/cpp/async/

CompletionQueue is an event queue. "event" here can be the completion of a request data reception or the expiry of an alarm(timer), etc. (basically, the completion of any asynchronous operation.)

Using [the official gRPC asynchronous APIs example](https://github.com/grpc/grpc/blob/v1.38.0/examples/cpp/helloworld/greeter_async_server.cc) as example, focus on the `CallData` class and `HandleRpcs()`:

``` c++
  void HandleRpcs() {
    // Spawn a new CallData instance to serve new clients.
    new CallData(&service_, cq_.get());
    void* tag;  // uniquely identifies a request.
    bool ok;
    while (true) {
      // Block waiting to read the next event from the completion queue. The
      // event is uniquely identified by its tag, which in this case is the
      // memory address of a CallData instance.
      // The return value of Next should always be checked. This return value
      // tells us whether there is any kind of event or cq_ is shutting down.
      GPR_ASSERT(cq_->Next(&tag, &ok));
      GPR_ASSERT(ok);
      static_cast<CallData*>(tag)->Proceed();
    }
  }
```

HandleRpcs() is the main loop of the server. It's an infinite loop which continuously gets the next event from the completion queue by using `cq->Next()` , and calls it's `Proceed()` method (our custom method to process client request of different states).

The `CallData` class (instance of which represents a complete processing cycle of a client request):

```c++
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
        // Make this instance progress to the PROCESS state.
        status_ = PROCESS;

        // As part of the initial CREATE state, we *request* that the system
        // start processing SayHello requests. In this request, "this" acts are
        // the tag uniquely identifying the request (so that different CallData
        // instances can serve different requests concurrently), in this case
        // the memory address of this CallData instance.
        service_->RequestSayHello(&ctx_, &request_, &responder_, cq_, cq_,
                                  this);
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
        status_ = FINISH;
        responder_.Finish(reply_, Status::OK, this);
      } else {
        GPR_ASSERT(status_ == FINISH);
        // Once in the FINISH state, deallocate ourselves (CallData).
        delete this;
      }
    }

   private:
    // The means of communication with the gRPC runtime for an asynchronous
    // server.
    Greeter::AsyncService* service_;
    // The producer-consumer queue where for asynchronous server notifications.
    ServerCompletionQueue* cq_;
    // Context for the rpc, allowing to tweak aspects of it such as the use
    // of compression, authentication, as well as to send metadata back to the
    // client.
    ServerContext ctx_;

    // What we get from the client.
    HelloRequest request_;
    // What we send back to the client.
    HelloReply reply_;

    // The means to get back to the client.
    ServerAsyncResponseWriter<HelloReply> responder_;

    // Let's implement a tiny state machine with the following states.
    enum CallStatus { CREATE, PROCESS, FINISH };
    CallStatus status_;  // The current serving state.
  };
```

As we can see, a `CallData` has three states: CREATE, PROCESS and FINISH.

## A request routine looks like this:

0. At startup, preallocates *one* CallData for a future incoming client.
1. During the construction of that CallData object, `service_->RequestSayHello(&ctx_, &request_, &responder_, cq_, cq_, this)` gets called, which tells gRPC to prepare for the reception of *exactly one* `SayHello` request.  
At this point we don't know where the request will come from or when it will come, we are just telling gRPC that we are ready to process when one actually arrives, and let gRPC notice us when it happens.  
Arguments to `RequestSayHello` tells gRPC where to put the context, request body and responder of the request after receiving one, as well as which completion queue to use for the notice and what tags should be attached to the notice event (in this case, `this` is used as the tag).
2. `HandleRpcs()` blocks on `cq->Next()`. Waiting for an event to occur.

__some time later....__

3. client makes a `SayHello` request to the server, **gRPC starts receiving and decoding that request. (IO operation)**

__some time later....__

4. gRPC have finished receiving the request. It puts the request body into the `request_` field of the CallData object (via the pointer supplied earlier), then creates an event (with `the pointer to the CallData object` as tag, as asked earlier by the last argument to `RequestSayHello`). gRPC then **puts that event into the completion queue `cq_`**.
5. **The loop in `HandleRpcs()` received the event**(the previously blocked call to `cq->Next()` returns now), calls `CallData::Proceed()` to process the request.
6. `status_` of the CallData is `PROCESS`, so it does the following:  
	6.1. Creates a new CallData object, so that new client requests after this one can be processed.  
	6.2. Generates the reply for the request, tells gRPC we have finished processing and please send the reply back to the client.  
	6.3 **gRPC starts transmission of the reply. (IO operation)**  
	6.4 The loop in `HandleRpcs()` goes into the next iteration and blocks on `cq->Next()` again, waiting for a new event to occur.  

__some time later....__

7. gRPC have finished transmission of the reply and tells us that by again **putting an event into the completion queue** with a pointer to CallData as the tag.
8. `cq->Next()` **receives the event and returns**, `CallData::Proceed()` deallocates the CallData object (by using `delete this;`). `HandleRpcs()` loops and blocks on `cq->Next()` again, waiting for a new event.

It might look like the process is largely the same as synchonous API, just with extra access to the completion queue. However, by doing it this way, at each and every `some time later....` (usually is waiting for IO operation to complete or waiting for a request to occur), `cq->Next()` can actually receive operation completion events not only for this request, but for other requests as well.

**So if a new request come in while the first request is, let's say, waiting for the transmission of reply data to finish, `cq->Next()` will get the event emitted by the new request, and starts the processing of the new request immediately and concurrently, instead of waiting for the first request to finish its transmission.**

Synchonous API, on the other hand, will always wait for the **full completion** of one request (from start receiving to finish replying) before even starting the receiving of another one. This meant near 0% CPU utilization while receiving request body data and sending back reply data (IO operations). Precious CPU time that could have been used to process other requests are wasted on just waiting.

This is really bad since if a client with a bad internet connection (100ms round-trip) sent a request to the server, we will have to spend at least 200ms for every request from this client just on actively waiting for TCP transmission to finish. That would bring our server performance down to only ~5 requests per second.

Whereas if we are using asynchonous API, we just don't actively wait for anything. We tell gRPC: "please send this data to the client, but we will not wait for you to finish here. Instead, just put a little letter to the completion queue when you are done, and we'll check it later." and move on to process other requests.

# Related information

You can see how a simple server is written for both [synchronous APIs](https://github.com/grpc/grpc/blob/v1.38.0/examples/cpp/helloworld/greeter_server.cc) and [asynchronous APIs](https://github.com/grpc/grpc/blob/v1.38.0/examples/cpp/helloworld/greeter_async_server.cc)

## Best performance practices

The best performance practice suggested by the [gRPC C++ Performance Nodes](https://grpc.github.io/grpc/cpp/md_doc_cpp_perf_notes.html) is to spawn the amount of threads equal to your CPU cores count, and use one CompletionQueue per thread.

