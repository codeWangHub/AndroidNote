# Android Binder系统

Binder系统是Google公司为Adnroid更加高效的进程间通信而设计添加的Linux驱动程序。

[TOC]

## 1. Binder 进程间通信原理

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

## 2. Binder通信模型 

### \<1\> Binder应用模型

Binder系统中有三个不同的“职位”。

- `Server`  : 服务器，提供服务。把自己的服务注册给`ServiceManager。`;
- `Client` : 客户端，使用服务，从`ServiceManager`获得服务，使用服务。
- `ServiceManager` : 服务器，管理服务。

**注意这是站在用户的角度！！，如果站在Binder驱动的角度就不是这个样子，后面会讲。**

![Binder机制通信模型](G:\work\android\android_base\binder\Binder机制通信模型.jpg)

可以看到`Client`要使用`Server`的某个服务，前提是这个服务在`ServiceManager`中注册过，`Client`知道要调用服务的哪个函数（函数编号指定）。综上`ServiceManager`起到中间桥梁的作用，所以`ServiceManager`是最先运行的。然后是`server`注册服务，最后是`client`使用服务。

接下来我们越过Binder驱动的源码，看看上层应用程序是如何使用binder系统进行通信的，分析它的源代码。

###  \<2\> binder通信应用程序情景分析

 在Android源代码中有关于Binder通信的例子程序: `frameworks\native\cmds\servicemanager`

![Android自带测试程序](G:\work\android\android_base\binder\Android自带测试程序.jpg)

我们先看看`sevice_manager.c`

```c
#include "binder.h"

uint32_t svcmgr_handle;

int main(int argc, char **argv)
{
	struct binder_state *bs;
  	bs = binder_open(128*1024);    // 打开binder驱动
    // 告诉binder驱动自己就是serviceManager
  	binder_become_context_manager(bs);  
  
    svcmgr_handle = BINDER_SERVICE_MANAGER;
    binder_loop(bs, svcmgr_handler);   /* 循环处理 */
}
```

- 解析 `struct binder_state*  binder_opne(128*1024)`方法

  ```c
  struct binder_state
  {
      int fd;         /* 打开binder驱动的句柄 */
      void *mapped;   /* mapp 内核空间内存的地址 */
      size_t mapsize; /* mmap 的大小 */
  };

  struct binder_state *binder_open(size_t mapsize)
  {
      struct binder_state *bs;
      struct binder_version vers;   

      bs = malloc(sizeof(*bs));    // 分配 binder_state 空间    
      bs->fd = open("/dev/binder", O_RDWR); // 打开 binder 驱动  
      ioctl(bs->fd, BINDER_VERSION, &vers);  // 获得 binder 版本号
       
      bs->mapsize = mapsize;  // 设置 mappsize
      /* mmap，从第一个参数NULL可以知道，mmap的内存是任意的 */
      bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
      return bs;
  }
  ```

- 解析 `int binder_become_context_manager(struct binder_state *)`

  ```c
  int binder_become_context_manager(struct binder_state *bs)
  {   // 直接调用了ioctl与binder驱动通信。
      return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
  }
  ```

- 解析 `binder_loop()`

  ```c
  void binder_loop(struct binder_state *bs, binder_handler func)
  {
      int res;
      struct binder_write_read bwr;
      uint32_t readbuf[32];

      bwr.write_size = 0;
      bwr.write_consumed = 0;
      bwr.write_buffer = 0;

      readbuf[0] = BC_ENTER_LOOPER;
      binder_write(bs, readbuf, sizeof(uint32_t));

      for (;;) {
          bwr.read_size = sizeof(readbuf);
          bwr.read_consumed = 0;
          bwr.read_buffer = (uintptr_t) readbuf;

          ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        
          binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
      }
  }
  ```

  ​

  ​



