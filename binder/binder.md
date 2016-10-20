[TOC]
#图解Android - Binder 和 Service
在 Zygote启动过程 一文中我们说道，Zygote一生中最重要的一件事就是生下了 System Server 这个大儿子，System Server 担负着提供系统 Service的重任，在深入了解这些Service 之前，我们首先要了解 什么是Service？它的工作原理是什么？

##1. Service是什么？

简单来说，Service就是提供服务的代码，这些代码最终体现为一个个的接口函数，所以，Service就是实现一组函数的对象，通常也称为组件。Android 的Service 有以下一些特点：

1. 请求Service服务的代码(Client)  和 Service本身(Server) 不在一个线程，很多情况下不在一个进程内。跨进程的服务称为远端(Remote)服务，跨进程的调用称为IPC。通常应用程序通过代理(Proxy)对象来访问远端的Service。
2. Service 可以运行在native 端(C/C++)，也可以运行在Java 端。同样，Proxy 可以从native 端访问Java Service, 也可以从Java端访问native service， 也就是说，service的访问与语言无关。
3. Android里大部分的跨进程的IPC都是基于Binder实现。
4. Proxy 通过 Interface 类定义的接口访问Server端代码。
5. Service可以分为匿名和具名Service. 前者没有注册到ServiceManager, 应用无法通过名字获取到访问该服务的Proxy对象。
6. Service通常在后台线程执行（相对于前台的Activity), 但Service不等同于Thread，Service可以运行在多个Thread上，一般这些Thread称为 Binder Thread.
  
要了解Service，我们得先从 Binder 入手。
##2.  Binder
 ![鸟图](/home/wuchuan/projeck/myProject/test/Notification-demo/binder/08022753-931f87a29bac4189a9aaa3877168963e.png)
先给一张Binder相关的类图一瞰Binder全貌，从上面的类图（点击看大图）跟Binder大致有这么几部分：

1. Native 实现:  IBinder,  BBinder, BpBinder, IPCThread, ProcessState, IInterface, etc 
2. Java 实现:  IBinder, Binder, BinderProxy, Stub, Proxy 
3. Binder Driver: binder_proc, binder_thread, binder_node, etc

我们将分别对这三部分进行详细的分析，首先从中间的Native实现开始。
通常来说，接口是分析代码的入口，Android中'I' 打头的类统统是接口类（C++里就是抽象类）, 自然，分析Binder就得先从IBinder下手。先看看他的定义。
```c++
class IBinder : public virtual RefBase
{
public:
    ...
    virtual sp<IInterface>  queryLocalInterface(const String16& descriptor); //返回一个IInterface对象
    ...
    virtual const String16& getInterfaceDescriptor() const = 0; 
    virtual bool            isBinderAlive() const = 0;
    virtual status_t        pingBinder() = 0;
    virtual status_t        dump(int fd, const Vector<String16>& args) = 0;
    virtual status_t        transact(   uint32_t code,
                                        const Parcel& data,
                                        Parcel* reply,
                                        uint32_t flags = 0) = 0;
    virtual status_t        linkToDeath(const sp<DeathRecipient>& recipient,
                                        void* cookie = NULL,
                                        uint32_t flags = 0) = 0;
    virtual status_t        unlinkToDeath(  const wp<DeathRecipient>& recipient,
                                            void* cookie = NULL,
                                            uint32_t flags = 0,
                                            wp<DeathRecipient>* outRecipient = NULL) = 0;
    ...
    virtual BBinder*        localBinder();  //返回一个BBinder对象
    virtual BpBinder*       remoteBinder(); //返回一个BpBinder对象
};
```
