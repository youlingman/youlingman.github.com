---
title: android的handler机制
date: 2018-03-26 22:22:22
tags: [android]
---

在Android应用里，谷歌只允许在主线程即UI线程更新UI，在这个主线程里做一些耗时的操作会导致[ANR](https://developer.android.com/topic/performance/vitals/anr.html)，于是一些耗时间的操作，如网络请求，就需要搬到额外线程去执行，而耗时操作之后往往又需要返回主/UI线程去更新UI。针对这个场景Android提供了一些组件给开发者使用，包括Handler、HandlerThread和AsyncTask，后两者其实都是基于Handler的实现。

<!--more-->

按官方[reference](https://developer.android.com/reference/android/os/Handler.html)，使用Handler可以向指定线程的MessageQueue发送Message或者Runnable对象。Handler机制包括Handler、Looper、MessageQueue、Message这几个组件。

- Looper：ThreadLocal变量，内部维护一个MessageQueue（自然也是ThreadLocal），在当前线程初始化looper并调用loop方法可启动对消息队列的轮询，使得当前线程可以接收对应Handler发送的消息并分发给对应Handler处理；
- MessageQueue：对消息队列的封装；
- Handler：一个用来向特定线程发送消息的‘句柄’，需与Looper绑定，持有Looper和MessageQueue引用，可以向对应Looper内的MessageQueue发送消息（enqueueMessage），同时也可以处理消息（handleMessage）；
- Message：对消息的封装，内含发送该消息的sendHandler的引用，因此轮询得到的新消息可以直接传给sendHandler去处理，除此之外Message类还维护了一个空闲message池，通过Message.obtain方法可利用该message池的资源；

可见，Handler机制是一种典型的基于消息队列与生产者/消费者模式的消息传递机制。对已开启Looper的线程集来说，每个线程维护并轮询自身的消息队列，而每个线程可以向任意其它线程发送消息，有点无中心消息网络的意思。当然Android提供多种方式获取主线程的Handler/Looper，以方便切换到UI线程执行UI更新任务。

回到开始线程切换的需求场景，利用Handler进行线程切换，可以概括成以下几步。

1. 如果需要从其它线程切换到线程A去执行对应逻辑，首先要获取到该线程的Looper，当然该线程必须是已经启动过Looper的；
2. 然后实例化一个Handler绑到线程A的Looper，就可以利用该Handler往线程A的消息队列写入消息；
3. 线程A的Looper轮询得到对应写入消息，分发回该target Handler的dispatchMessage方法进行处理；
4. 需要切换线程执行的逻辑可以写在发送的Runnable消息内、Handler的callback参数内、Handler的handleMessage方法内，执行优先级由前往后；

最后看看核心的MessageQueue的一些设计。

### 关于队列实现

MessageQueue中没有使用显式的队列容器去保存消息，‘消息队列’是通过Message自带的next变量维护成一个Message链表来实现的。写入新消息时按消息延时的执行时序确定插入位置。

### 关于轮询

从源码来看，轮询依次阻塞在Looper.loop->MessageQueue.next->nativePollOnce，而写入的调用顺序则是Handler.enqueueMessage->MessageQueue.enqueueMessage->nativeWake，这里nativePollOnce的实现依赖于Linux底层的IO监听系统调用epoll来监听写入动作，而nativeWake则是对管道Pipe写入字符来唤醒epoll。

### 关于并发读/写

在MessageQueue.next和MessageQueue.enqueueMessage中使用MessageQueue实例对象锁synchronized控制并发读、并发写和并发读写。

### 关于延时发送

唤起轮询后，MessageQueue.next中会判断链表头消息的when是否已到时间，如果没到时间则设置epoll的timeout并继续监听。