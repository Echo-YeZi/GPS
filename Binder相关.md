# Binder

## 1. 预备知识

#### 各种IPC方式数据拷贝次数

IPC | 数据拷贝次数
-|-
共享内存|0
Binder|1
Socket/管道/消息队列|2

1. **socket**

   > 通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信
   >
   > 采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区
   >
   > 存在被伪造身份的安全风险

2. **共享内存**

   > 无需拷贝，但控制复杂，难以使用

3. **Binder**

   > 基于Client-Server通信模式，传输过程只需一次拷贝，为发送发添加UID/PID身份，既支持实名Binder也支持匿名Binder，安全性高



## 2. 简介

**Binder使用Client-Server通信方式**

- 一个进程作为Server提供诸如视频/音频解码，视频捕获，地址本查询，网络连接等服务

- 多个进程作为Client向Server发起服务请求，获得所需要的服务

**Client-Server通信据条件**

- server必须有确定的访问接入点或者说地址来接受Client的请求，并且Client可以通过某种途径获知Server的地址
- 制定Command-Reply协议来传输数据。例如在网络通信中Server的访问接入点就是Server主机的IP地址+端口号，传输协议为TCP协议
  - 对Binder而言，Binder可以看成Server提供的实现某个特定服务的访问接入点， Client通过这个‘地址’向Server发送请求来使用该服务；
  - 对Client而言，Binder可以看成是通向Server的管道入口，要想和某个Server通信首先必须建立这个管道并获得管道入口

**Binder使用面向对象的思想**

- Binder是一个实体位于Server中的对象，该对象提供了一套方法用以实现对服务的请求，就象类的成员函数
- 从通信的角度看，Client中的Binder也可以看作是Server Binder的‘代理’，在本地代表远端Server为Client提供服务



## 3.模型

> Binder框架定义了四个角色：Server，Client，ServiceManager（以后简称SMgr）以及Binder驱动。
>
> - 其中Server，Client，SMgr运行于用户空间，驱动运行于内核空间。
> - 四个角色的关系和互联网类似：Server是服务器，Client是客户终端，SMgr是域名服务器（DNS），驱动是路由器