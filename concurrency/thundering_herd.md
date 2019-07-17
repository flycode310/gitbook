# 惊群效应

## accept惊群

### 服务器网络模型和惊群

传统的服务器使用“listen-accept-创建通信socket”完成客户端的一次请求服务。在高并发服务模型中，服务器创建很多进程-单线程（比如apache mpm）或者n进程：m线程比例创建服务线程（比如nginx event）。机器上运行着不等数量的服务进程或线程。这些进程监听着同一个socket。这个socket是和客户端通信的唯一地址。服务器父子进程或者多线程模型都accept该socket，有几率同时调用accept。当一个请求进来，accept同时唤醒等待socket的多个进程，但是只有一个进程能accept到新的socket，其他进程accept不到任何东西，只好继续回到accept流程。这就是惊群效应。如果使用的是select/epoll+accept，则把惊群提前到了select/epoll这一步，多个进程只有一个进程能acxept到连接，因为是非阻塞socket，其他进程返回EAGAIN。

### accept惊群的解决

所有监听同一个socket的进程在内核中中都会被放在这个socket的wait queue中。当一个tcp socket有IO事件变化，都会产生一个wake_up_interruptible()。该系统调用会唤醒wait queue的所有进程。所以修复linux内核的办法是只唤醒一个进程，比如说替换wake函数为wake_one_interruptoble()。

没有开启reuse选项的socket只有一个wait queue，假设在开启了socket REUSE_PORT选项，内核中为每个进程分配了单独的accept wait queue，每次唤醒wait queue只唤醒有请求的进程。协议栈将socket请求均匀分配给每个accept wait queue。reuse部分解决了惊群问题，但是本身存在一些缺点或bug，比如REUSE实现是根据客户端ip端口实现哈希，对同一个客户请求哈希到同一个服务器进程，但是没有实现一致性哈希。在进程数量扩展新的进程，由于缺少一致性哈希，当listen socket的数目发生变化（比如新服务上线、已存在服务终止）的时候，根据SO_REUSEPORT的路由算法，在客户端和服务端正在进行三次握手的阶段，最终的ACK可能不能正确送达到对应的socket，导致客户端连接发生Connection Reset，所以有些请求会握手失败。

## epoll惊群

在一个高并发的服务器模型中，每秒accept的连接数很多。accept成为一个占用cpu很高的系统调用。考虑使用多进程来accept。select由于可扩展性能比如epoll，select遍历所有socket，select对每次操作都是要循环遍历所有的fd，所以在高并发场景下，select性能差。在高并发场景epoll使用场景更多。

liunx 4.5内核在epoll已经新增了EPOLL_EXCLUSIVE选项，在多个进程同时监听同一个socket，只有一个被唤醒。

## nginx惊群

## 线程池惊群

## golang中惊群的解决

```go
func (fd *netFD) accept() (netfd *netFD, err error) {
   ／／在这里序列化accept，避免惊群效应
  if err := fd.readLock(); err != nil {
        return nil, er
    }
    defer fd.readUnlock()
    ......

    for {
        s, rsa, err = accept(fd.sysfd)
        if err != nil {
            if err == syscall.EAGAIN {
                ／／tcp还没三次握手成功，阻塞读直到成功，同时调度控制权下放给gorontine
                if err = fd.pd.WaitRead(); err == nil {
                    continue
                }
            } else if err == syscall.ECONNABORTED {
                ／／被对端关闭
                continue
            }
        }
        break
    }

    netfd, err = newFD(s, fd.family, fd.sotype, fd.net)
    ......

    //fd添加到epoll队列中
    err = netfd.init()
    ......
    lsa, _ := syscall.Getsockname(netfd.sysfd)
    netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
    return netfd, nil
}
```



## reference

[高并发中的惊群效应](https://blog.csdn.net/second60/article/details/81252106)

[惊群效应](https://cloud.tencent.com/developer/article/1340628)

[linux惊群效应](https://www.cnblogs.com/zafu/p/8251849.html)