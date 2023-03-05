在网络开发模型中，有一种非常易于开发同学使用的方式，那就是同步阻塞的网络 IO（在 Java 中习惯叫 BIO）。
例如我们想请求服务器上的一段数据，那么 C 语言的一段代码 demo 大概是下面这样：
```c
int main()
{
    int sk = socket(AF_INET, SOCK_STREAM, 0);
    connect(sk, ...)
    recv(sk, ...)
}
```

但是在高并发的服务器开发中，这种网络 IO 的性能奇差。因为

- 1.进程在 recv 的时候大概率会被阻塞掉，导致一次进程切换
- 2.当连接上数据就绪的时候进程又会被唤醒，又是一次进程切换
- 3.一个进程同时只能等待一条连接，如果有很多并发，则需要很多进程

如果用一句话来概括，那就是：**同步阻塞网络 IO 是高性能网络开发路上的绊脚石！** 俗话说得好，知己知彼方能百战百胜。所以我们今天先不讲优化，只深入分析同步阻塞网络 IO 的内部实现。
在上面的 demo 中虽然只是简单的两三行代码，但实际上用户进程和内核配合做了非常多的工作。先是用户进程发起创建 socket 的指令，然后切换到内核态完成了内核对象的初始化。接下来 Linux 在数据包的接收上，是硬中断和 ksoftirqd 进程在进行处理。当 ksoftirqd 进程处理完以后，再通知到相关的用户进程。
从用户进程创建 socket，到一个网络包抵达网卡到被用户进程接收到，总体上的流程图如下：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26185941/1651675579458-98c752d5-a4e7-4320-82be-5d9d15abbe82.png#clientId=u305f7059-7436-4&from=paste&height=519&id=u3f62df61&name=image.png&originHeight=519&originWidth=570&originalType=binary&ratio=1&rotation=0&showTitle=false&size=116722&status=done&style=none&taskId=uca7af34c-fcf7-4fc5-a0e0-99c219e0cbb&title=&width=570)
我们今天用图解加源码分析的方式来详细拆解一下上面的每一个步骤，来看一下在内核里是它们是怎么实现的。阅读完本文，你将深刻地理解在同步阻塞的网络 IO 性能低下的原因！

![image.png](https://cdn.nlark.com/yuque/0/2022/png/26185941/1651675592221-f687cc29-da71-43bc-bf77-bfa5e1feb4a3.png#clientId=u305f7059-7436-4&from=paste&height=513&id=u3a2ff390&name=image.png&originHeight=513&originWidth=564&originalType=binary&ratio=1&rotation=0&showTitle=false&size=97831&status=done&style=none&taskId=ua36a5b75-47ea-4bae-9daa-202d4401e8e&title=&width=564)

我们来翻翻源码，看下上面的结构是如何被创造出来的。
```c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
 ......
 retval = sock_create(family, type, protocol, &sock);
}
```
sock_create 是创建 socket 的主要位置。其中 sock_create 又调用了 __sock_create。
```c
//file:net/socket.c
int __sock_create(struct net *net, int family, int type, int protocol,
    struct socket **res, int kern)
{
 struct socket *sock;
 const struct net_proto_family *pf;

 ......

 //分配 socket 对象
 sock = sock_alloc();

 //获得每个协议族的操作表
 pf = rcu_dereference(net_families[family]);

 //调用每个协议族的创建函数， 对于 AF_INET 对应的是
 err = pf->create(net, sock, protocol, kern);
}
```
在 __sock_create 里，首先调用 sock_alloc 来分配一个 struct sock 对象。接着在获取协议族的操作函数表，并调用其 create 方法。对于 AF_INET 协议族来说，执行到的是 inet_create 方法。
```c
//file:net/ipv4/af_inet.c
tatic int inet_create(struct net *net, struct socket *sock, int protocol,
         int kern)
{
 struct sock *sk;

 //查找对应的协议，对于TCP SOCK_STREAM 就是获取到了
 //static struct inet_protosw inetsw_array[] =
    //{
 //    {
 //     .type =       SOCK_STREAM,
 //     .protocol =   IPPROTO_TCP,
 //     .prot =       &tcp_prot,
 //     .ops =        &inet_stream_ops,
 //     .no_check =   0,
 //     .flags =      INET_PROTOSW_PERMANENT |
 //            INET_PROTOSW_ICSK,
 //    },
 //}
    list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {

 //将 inet_stream_ops 赋到 socket->ops 上 
 sock->ops = answer->ops;

 //获得 tcp_prot
 answer_prot = answer->prot;

 //分配 sock 对象， 并把 tcp_prot 赋到 sock->sk_prot 上
 sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot);

 //对 sock 对象进行初始化
 sock_init_data(sock, sk);
}
```
在 inet_create 中，根据类型 SOCK_STREAM 查找到对于 tcp 定义的操作方法实现集合 inet_stream_ops 和 tcp_prot。并把它们分别设置到 socket->ops 和 sock->sk_prot 上。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26185941/1651675671006-53aaefaa-55ea-4737-8587-526dd6fdc523.png#clientId=u305f7059-7436-4&from=paste&height=300&id=u9878d243&name=image.png&originHeight=300&originWidth=551&originalType=binary&ratio=1&rotation=0&showTitle=false&size=55771&status=done&style=none&taskId=uffe3fb1e-4641-426a-af76-9736db91871&title=&width=551)
我们再往下看到了 sock_init_data。在这个方法中将 sock 中的 sk_data_ready 函数指针进行了初始化，设置为默认 sock_def_readable()。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26185941/1651675682828-36c3c96f-30d8-47e3-aba5-08520ad9a347.png#clientId=u305f7059-7436-4&from=paste&height=118&id=u24b28780&name=image.png&originHeight=118&originWidth=603&originalType=binary&ratio=1&rotation=0&showTitle=false&size=24109&status=done&style=none&taskId=u587ac548-8aed-4862-b805-cab744193cb&title=&width=603)
