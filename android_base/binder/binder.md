# Android Binder系统

Binder系统是Google公司为Adnroid更加高效的进程间通信而设计添加的Linux驱动程序。

[TOC]

### 1. Binder 进程间通信原理

- Unix 原生进程间通信原理

  Unix环境下的进程间通信方式有很多，比如常见的信号、信号量、管道、套接字等等。无论是什么通信手段其内部最基本的原理是差不多的，如图所示。

  ![Unix进程间通信原理](G:\work\android\android_base\binder\Unix进程间通信原理.jpg)

  可以看到，数据从进程A发送到进程B涉及到两次数据的拷贝。当然进程间通信手段还有一种叫做共享内存他是在内核中分配一块内存，两个进程都mmap这块内存。但是这种方式有两个问题，一是多线程不安全，二是一个进程写完数据还要利用其他通信手段告知另一个进程。


- Binder驱动进程间通信原理

  ![Binder通信原理](G:\work\android\android_base\binder\Binder通信原理.jpg)

  可以看到对局数据头部有两次拷贝，但是对于数据本身只有一次拷贝，通信效率大大提高。


- Binder机制的优势

  在[Linux](http://lib.csdn.net/base/linux)中使用的IPC通信机制如下：

  - 传统IPC：`无名pipe, signal, trace, 有名管道`
  - AT&T Unix 系统V：`共享内存，信号灯，消息队列`
  - BSD Unix：`Socket`

  Binder相比较这些通信方式的优点：

  - 采用C/S的通信模式。而在linux通信机制中，目前只有socket支持C/S的通信模式，但socket有其劣势，具体参看第二条。
  - 有更好的传输性能。对比于Linux的通信机制，
    - `socket`：是一个通用接口，导致其传输效率低，开销大；
    - `管道和消息队列`：因为采用存储转发方式，所以至少需要拷贝2次数据，效率低；
    - `共享内存`：虽然在传输时没有拷贝数据，但其控制机制复杂（比如跨进程通信时，需获取对方进程的pid，得多种机制协同操作）。
  - 安全性更高。Linux的IPC机制在本身的实现中，并没有安全措施，得依赖上层协议来进行安全控制。而Binder机制的UID/PID是由Binder机制本身在内核空间添加身份标识，安全性高；并且Binder可以建立私有通道，这是linux的通信机制所无法实现的（Linux访问的接入点是开放的）。
  - 另外一个优点是对用户来说，通过binder屏蔽了client的调用server的隔阂，client端函数的名字、参数和返回值和server的方法一模一样，对用户来说犹如就在本地（也可以做得不一样），这样的体验或许其他ipc方式也可以实现，但binder出生那天就是为此而生。

### 2. Binder通信模型 

Binder系统中有三个不同的“职位”。

- `Server`  : 服务器，提供服务。把自己的服务注册给`ServiceManager。`;
- `Client` : 客户端，使用服务，从`ServiceManager`获得服务，使用服务。
- `ServiceManager` : 服务器，管理服务。

**注意这是站在用户的角度！！，如果站在Binder驱动的角度就不是这个样子，后面会讲。**







