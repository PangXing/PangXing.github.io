---
layout: post_layout
title:  Protobuf RPC的原生服务框架
time:   2022年06月24日 星期四
location: 北京
pulished: true
---

市面上基于Google Protobuf RPC 机制实现的RPC优秀框架众多，如brpc、sofa-pbrpc、与grpc等。但是这些框架涵盖了许多复杂逻辑，却隐藏了基于Probobuf RPC实现RPC的主线。本文主要介绍一下Google Protobuf RPC 机制，并实现一套简单的RPC框架，有助于品读其他优秀的RPC框架源码。

## 1.Google Protobuf RPC 的原始框架提供了哪些器件？
通过一个经典的echo实例进行说明Google Protobuf 如何定义一个RPC服务，以及Protobuf RPC原始框架提供了哪些基础器件？
![](/assets/img/protobuf_rpc/echo_pb.png#pic_center=200x)
在.proto文件定义RPC服务接口EchoServer，并通过protoc工具根据你需要的语言版本，分别生成了服务端Service接口EchoServer与客户端存根接口EchoServer_Stub。其中客户端中的通信是调用了RpcChannel接口实现，而服务端与客户端交互过程又引入了RpcController接口进行干预。由此总结一下Protobuf RPC 原始框架定义的这些基础器件：
+ Serveric
  - Abstract base interface for protocol-buffer-based RPC services. Services themselves are abstract interfaces (implemented either by servers or as stubs), but they subclass this base interface. The methods of this interface can be used to call the methods of the Service without knowing its exact type at compile time (analogous to Reflection).
  - 服务端Service接口EchoServe，定制服务端业务逻辑，其底层函数CallMethod中实现了Method映射，用户进行业务逻辑二次开发时，只需重载业务Method即可（如实例重载Echo函数）。其中需要注意Echo函数最后，调用了done->Run()，作用是实现服务端数据打包、序列化、与使用指定的传输协议进行传输等功能。Protobuf RPC 原生框架并没有真正实现，而是回调注册在Closure中的实现函数，如grpc框架注册后实现方案是 PB序列化+http2协议传输。
  - 客户端存根接口EchoServer_Stub，定制客户端业务逻辑，其底层实际调用RpcChannel接口，但Protobuf RPC 原生框架的RpcChannel是纯虚类。

+ RpcChannel
  - Abstract interface for an RPC channel. An RpcChannel represents a communication line to a Service which can be used to call that Service’s methods. The Service may be running on another machine. Normally, you should not call an RpcChannel directly, but instead construct a stub Service wrapping it.
  - Channel可以理解为一条客户端项服务端发起的通讯管道，其中CallMethod需要实现数据序列化反序列化与发送数据等功能。但Protobuf RPC 原生框架并没有实现，而是留给二次开发进行定制。

+ RpcController
  - An RpcController mediates a single method call. The primary purpose of the controller is to provide a way to manipulate settings specific to the RPC implementation and to find out about RPC-level errors. The methods provided by the RpcController interface are intended to be a “least common denominator” set of features which we expect all implementations to support. Specific implementations may provide more advanced features (e.g. deadline propagation).
  - RPCController作为RPC方法的入参，是方便获取或设置服务参数，从而影响服务。如客户端传入一个RpcController，其作用包括控制请求超时时间、获得重试次数、设置压缩类型与获得服务状态（失败时获取错误信息）等。服务端可获得客户端地址，并设置服务状态等。
  注意，RpcController用于控制单次请求，因此每次RPC请求（无论同步还是异步）都需要一个RpcController，在本次请求完成前，其不得释放或者重用。请求完成后，该RpcController可以使用Reset()方法恢复初始状态以进行重用，不同接口有不同的使用时间范围，只有在规定的时间范围内使用，接口才能返回正确结果，否则行为是不确定的。    
·
## 2.基于Protobuf RPC的可运行的RPC框架需要做些什么？
![](/assets/img/protobuf_rpc/pb_rpc.png#pic_center=200x)
通过上面的分析，我们知道了Protobuf RPC 原生框架只是提供了一个运行框架，涵盖了Service、RpcChannel、RPCcontroller抽象接口，需要开发者进行二次开发实现RPCChannelImpl、RPCControllerImpl与服务侧的callback函数注册，主要功能包括：
  - 数据的序列化与反序列化，可使用Protobuf 或 Json等；
  - 数据底层传输协议，如 Socket 、Http2 或 baidu_std等；
  - 服务Service的管理，如Service的注册与映射、服务治理等。

## 3. 可运行的RPC框架如何实现？
![](/assets/img/protobuf_rpc/simple_rpc.png#pic_center=200x)
  - RpcServerImpl 服务端服务管理类，包括了Service的注册函数RegisterService、response数据序列化与传输函数OnCallbackDone（注册回调） 与 服务监听与Service映射函数Start；
  - RPCChannel 客户端管道类，实现了CallMethod函数，其功能包括请求数据序列化，请求方法的编码与接收数据反序列化；
  - RpcController 交互信息控制类，包括了服务状态的管理与服务控制等功能。

