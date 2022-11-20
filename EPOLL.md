# EPOLL

> 参考：https://mp.weixin.qq.com/s/OmRdUgO1guMX76EdZn11UQ

## 二、epoll_create 实现

初始化eventpoll，并把它关联到当前进程的已打开文件列表中。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwolcxS62c1ZRibFc0NUVCJ46XI8tfQQDiakZP00NBK5ZWYCxY5t9mDjO2C4pKdOIp51ia0tx7NuW4LwA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwolcxS62c1ZRibFc0NUVCJ46RyK6lYFdDkVRH2xMBjFoeo0MKGzfMSGVFfbMuPtwAFt8w8FnQUIxeQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



eventpoll 这个结构体中的几个成员的含义如下：

- **wq：** 等待队列链表。软中断数据就绪的时候会通过 wq 来找到阻塞在 epoll 对象上的用户进程。
- **rbr：** 一棵红黑树。为了支持对海量连接的高效查找、插入和删除，eventpoll 内部使用了一棵红黑树。通过这棵树来管理用户进程下添加进来的所有 socket 连接。
- **rdllist：** 就绪的描述符的链表。当有的连接就绪的时候，内核会把就绪的连接放到 rdllist 链表里。这样应用进程只需要判断链表就能找出就绪进程，而不用去遍历整棵树。

## 三、epoll_ctl 添加 socket

- 1.分配一个红黑树节点对象 epitem
- 2.添加等待事件到 socket 的等待队列中，其回调函数是 ep_poll_callback
- 3.将 epitem 插入到 epoll 对象的红黑树里



通过 epoll_ctl 添加两个 socket 以后，这些内核数据结构最终在进程中的关系图大致如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwolcxS62c1ZRibFc0NUVCJ46UzHy4WvRpicUaNuqIibhPJcyRiacqeDZx0MX9sqkibCsIJDMbujC6IqA8w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



## 四、epoll_wait 等待接收

epoll_wait 做的事情不复杂，当它被调用时它观察 eventpoll->rdllist 链表里有没有数据即可。有数据就返回，没有数据就创建一个等待队列项，将其添加到 eventpoll 的等待队列上，然后把自己阻塞掉就完事。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwolcxS62c1ZRibFc0NUVCJ46DTCVLYKALXJpjB3Glp2bjPzVS3s8dASJeZ2wUfo0rlM2O7ic8y7Eib2g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



## 五、数据来啦

- socket->sock->sk_data_ready 设置的就绪处理函数是 **sock_def_readable**
- 在 socket 的等待队列项中，其回调函数是 **ep_poll_callback**。另外其 private 没有用了，指向的是空指针 null。
- 在 eventpoll 的等待队列项中，回调函数是 **default_wake_function**。其 private 指向的是等待该事件的用户进程。



### sock_def_readable

当 socket 上数据就绪时候，内核将以 sock_def_readable 这个函数为入口，找到 epoll_ctl 添加 socket 时在其上设置的回调函数 ep_poll_callback。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwolcxS62c1ZRibFc0NUVCJ46J3PXktkklzaGLPOJOJhTJUkfLMPz7uz1VXTFdmodxbm8d8RUP3lFeg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



### ep_poll_callback

在 ep_poll_callback 根据等待任务队列项上的额外的 base 指针可以找到 epitem， 进而也可以找到 eventpoll对象。

首先它做的第一件事就是**把自己的 epitem 添加到 epoll 的就绪队列中**。

接着它又会查看 eventpoll 对象上的等待队列里是否有等待项（epoll_wait 执行的时候会设置）。

如果没执行软中断的事情就做完了。如果有等待项，那就查找到等待项里设置的回调函数。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwolcxS62c1ZRibFc0NUVCJ46mAGU5iawBTWgB0sSk9oFaaxyRzGyCojhrmUgibMDC3b5LQBbdoRnM4UA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



### default_wake_function

在default_wake_function 中找到等待队列项里的进程描述符，然后唤醒之。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwolcxS62c1ZRibFc0NUVCJ469o9q43icrjlVG765wXDOrUgia2RdZ66K3thBHg1qSPibUu3NjlcgKMXqg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

等待队列项 curr->private 指针是在 epoll 对象上等待而被阻塞掉的进程。

将epoll_wait进程推入可运行队列，等待内核重新调度进程。然后epoll_wait对应的这个进程重新运行后，就从 schedule 恢复

当进程醒来后，继续从 epoll_wait 时暂停的代码继续执行。把 rdlist 中就绪的事件返回给用户进程



## 总结

epoll 的整个工作路程。

![图片](https://mmbiz.qpic.cn/mmbiz_png/BBjAFF4hcwolcxS62c1ZRibFc0NUVCJ46h2GrIOb4GbapWqATZwAALWXWH8505zthGzEEyiawU3TicRQgHMj0B0eg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)