---
layout: post  
title: NettyClient连接流程  
category: 技术相关  
tags: Netty  

---

## 初始化一个NioEventLoopGroup
* NioEventLoopGroup相当于一个底层的Reactor，可以对多个Channel进行数据处理，默认内部实现了一个SingleThreadExecuter数组，大小可设置，默认为CPU核心数*2 
* 关于Reactor和Proactor，前者为同步，后者为异步。Reactor情况下有数据可读后会通知handler，具体怎么读是由handler去实现，而Proactor则是把活干完了再通知handler，个人理解是由于底层bytebufer的连接性，因此netty采用的是Reactor模式
* 每个Channel被绑定到一个SingleThreadExecuter上，一个SingleThreadExecuter可以绑定多个Channel，也就是说，**每发起一个连接，当前连接所有的数据处理都是单线程执行**
* 当NioEventLoopGroup被Shutdown的时候，运行的所有Channel都被关闭
* ChannelGroup可以用来管理Channel，尤其是服务端的时候，可以一次性往所有活动的Channel中发消息，并且关闭的channel会自动从group中移除，无需求手动管理

## BootStrap实例化
* 必须设置EventLoopGroup和ChannelClass(ChannelClass用来关联ChannelFactory)
* 设置options，可以设置keepalive，nodelay等参数
* 设置handler，这里是用了一个匿名类来实现的，和闭包的调用比较类似，在发起连接时，BootStrap会对上面设置的一系列参数进行校验证，尤其是handler，**Decoder是不允许Sharable的,因为Decoder继承自ByteToMessageDecoder,而ByteToMessageDecoder内部有buf来缓存底Channel来的数据，如果Sharable了，一个Decoder会读到多个Channel接收的数据，encdoer是直接写到底层的buffer了，因此没有这个问题**，业务类的handler可以共享。数据到达业务类handler时已经和底层bufer脱离，不会出现bytebuffer串数据情况
* BootStrap可以通过Attr方法来将属性传递到Channel中，相当于用户自定义参数，这个对于一些每个连接独有的参数有用

## 发起连接
* 通过channelFactory生成一个Channel
* init Channel
* 向NioEventLoopGroup注册channel
* 所有操作都正常情况下，发起物理连接，并返回一个ChannelFuture
* ChannelFuture可以加入Listener用于处理连接操作的结果
* 上一步的Listener监听到连接成功后，可以在Channel的CloseFuture中加中Listener用于监听断连事件

## 关于BootStrap中的sync方法

* 在发起连接后，可以调用sync方法一直等待服务端响应，调用这方法，当连接异常时会抛出异常，而Listener则不会，无论连接成功或失败都会以事件的方式通知到Listener