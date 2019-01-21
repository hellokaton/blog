---
layout: post
title: 零拷贝 - 用户态分析
tags: ["linux", "翻译"]
---

现在几乎所有人都听过 Linux 下的零拷贝技术，但我经常遇到对这个问题不能深入理解的人。所以我写了这篇文章，来深入研究这些问题。本文通过用户态程序的角度来看零拷贝，因此我有意忽略了内核级别的实现。

# 什么是 “零拷贝” ？

为了更好的理解这个问题，我们首先需要了解问题本身。来看一个网络服务的简单运行过程，在这个过程中将磁盘的文件读取到缓冲区，然后通过网络发送给客户端。下面是示例代码：

```c
read(file, tmp_buf, len); 
write(socket, tmp_buf, len);
```

这个例子看起来非常简单，你可能会认为只有两次系统调用不会产生太多的系统开销。实际上并非如此，在这两次调用之后，数据至少被拷贝了 4 次，同时还执行了很多次 *用户态/内核态* 的上下文切换。（实际上这个过程是非常复杂的，为了解释我尽可能保持简单）为了更好的理解这个过程，请查看下图中的上下文切换，图片上部分展示上下文切换过程，下部分展示拷贝操作。

![两次系统调用]({{ "/public/images/2019/01/sys_call_copy.jpg" | prepend: site.cdnurl }} "两次系统调用")

1. 程序调用 `read` 产生一次用户态到内核态的上下文切换。DMA 模块从磁盘读取文件内容，将其拷贝到内核空间的缓冲区，完成第 1 次拷贝。
2. 数据从内核缓冲区拷贝到用户空间缓冲区，之后系统调用 `read` 返回，这回导致从内核空间到用户空间的上下文切换。这个时候数据存储在用户空间的 `tmp_buf` 缓冲区内，可以后续的操作了。
3. 程序调用 `write` 产生一次用户态到内核态的上下文切换。数据从用户空间缓冲区被拷贝到内核空间缓冲区，完成第 3 次拷贝。但是这次数据存储在一个和 `socket` 相关的缓冲区中，而不是第一步的缓冲区。
4. `write` 调用返回，产生第 4 个上下文切换。第 4 次拷贝在 DMA 模块将数据从内核空间缓冲区传递至协议引擎的时候发生，这与我们的代码的执行是独立且异步发生的。你可能会疑惑：“为何要说是独立、异步？难道不是在 `write` 系统调用返回前数据已经被传送了？write 系统调用的返回，并不意味着传输成功——它甚至无法保证传输的开始。调用的返回，只是表明以太网驱动程序在其传输队列中有空位，并已经接受我们的数据用于传输。可能有众多的数据排在我们的数据之前。除非驱动程序或硬件采用优先级队列的方法，各组数据是依照FIFO的次序被传输的(上图中叉状的 DMA copy 表明这最后一次拷贝可以被延后)。

# mmap

如你所见，上面的数据拷贝非常多，我们可以减少一些重复拷贝来减少开销，提升性能。作为一名驱动程序开发人员，我的工作围绕着拥有先进特性的硬件展开。某些硬件支持完全绕开内存，将数据直接传送给其他设备。这个特性消除了系统内存中的数据副本，因此是一种很好的选择，但并不是所有的硬件都支持。此外，来自于硬盘的数据必须重新打包(地址连续)才能用于网络传输，这也引入了某些复杂性。为了减少开销，我们可以从消除内核缓冲区与用户缓冲区之间的拷贝开始。

减少数据拷贝的一种方法是将 `read` 调用改为 `mmap`。例如：

```c
tmp_buf = mmap(file, len); 
write(socket, tmp_buf, len);
```

为了方便你理解，请参考下图的过程。

![mmap调用]({{ "/public/images/2019/01/sys_mmap.jpg" | prepend: site.cdnurl }} "mmap 调用")

1. `mmap` 调用导致文件内容通过 DMA 模块拷贝到内核缓冲区。然后与用户进程共享缓冲区，这样不会在内核缓冲区和用户空间之间产生任何拷贝。
2. `write` 调用导致内核将数据从原始内核缓冲区拷贝到与 `socket` 关联的内核缓冲区中。
3. 第 3 次数据拷贝发生在 DMA 模块将数据从 `socket` 缓冲区传递给协议引擎时。

通过调用 `mmap` 而不是 `read`，我们已经将内核拷贝数据操作减半。当传输大量数据时，效果会非常好。然而，这种改进并非没有代价；使用 `mmap + write` 方式存在一些隐藏的陷阱。当内存中做文件映射后调用 `write`，与此同时另一个进程截断这个文件时。此时 `write` 调用的进程会收到一个 `SIGBUS` 中断信号，因为当前进程访问了非法内存地址。这个信号默认情况下会杀死当前进程并生成 `dump` 文件——而这对于网络服务器程序而言不是最期望的操作。有两种方式可用于解决该问题：

第一种方法是处理收到的 `SIGBUS` 信号，然后在处理程序中简单地调用 `return`。通过这样做，`write` 调用会返回它在被中断之前写入的字节数，并且将全局变量 `errno` 设置为成功。我认为这是一个治标不治本的解决方案。因为收到 `SIGBUS` 信号表示程序发生了严重的错误，我不推荐使用它作为解决方案。

第二种方式应用了文件租借（在Microsoft Windows系统中被称为“机会锁”)。这才是解劝前面问题的正确方式。通过对文件描述符执行租借，可以同内核就某个特定文件达成租约。从内核可以获得读/写租约。当另外一个进程试图将你正在传输的文件截断时，内核会向你的进程发送实时信号——RT_SIGNAL_LEASE。该信号通知你的进程，内核即将终止在该文件上你曾获得的租约。这样，在write调用访问非法内存地址、并被随后接收到的SIGBUS信号杀死之前，write系统调用就被RT_SIGNAL_LEASE信号中断了。write的返回值是在被中断前已写的字节数，全局变量errno设置为成功。下面是一段展示如何从内核获得租约的示例代码。

```c
if (fcntl(fd, F_SETSIG, RT_SIGNAL_LEASE) == -1) {
    perror("kernel lease set signal");
    return -1;
}
/* l_type can be F_RDLCK F_WRLCK */
if (fcntl(fd, F_SETLEASE, l_type)) {
    perror("kernel lease set type");
    return -1;
}
```

在对文件进行映射前，应该先获得租约，并在结束 `write` 操作后结束租约。这是通过在 `fcntl` 调用中指定租约类型为 `F_UNLCK` 来实现的。

# Sendfile

在内核的 2.1 版本中，引入了 `sendfile` 系统调用，目的是简化通过网络和两个本地文件之间的数据传输。`sendfile` 的引入不仅减少了数据拷贝，还减少了上下文切换。可以这样使用它：

```c
sendfile(socket, file, len);
```

同样的，为了理解起来方便，可以看下图的调用过程。

![sendfile代替读写]({{ "/public/images/2019/01/sys_sendfile.jpg" | prepend: site.cdnurl }} "sendfile 代替读写")

1. `sendfile` 调用会使得文件内容通过 DMA 模块拷贝到内核缓冲区。然后，内核将数据拷贝到与 `socket` 关联的内核缓冲区中。
2. 第 3 次拷贝发生在 DMA 模块将数据从内核 `socket` 缓冲区传递到协议引擎时。

你可能想问当我们使用 `sendfile` 调用传输文件时有另一个进程截断会发生什么？如果我们没有注册任何信号处理程序，`sendfile` 调用只会返回它在被中断之前传输的字节数，并且全局变量 `errno` 被设置为成功。

但是，如果我们在调用 `sendfile` 之前从内核获得了文件租约，那么行为和返回状态完全相同。我们会在`sendfile` 调用返回之前收到一个 `RT_SIGNAL_LEASE` 信号。

到目前为止，我们已经能够避免让内核产生多次拷贝，但我们还有一次拷贝。这可以避免吗？当然，在硬件的帮助下。为了避免内核完成的所有数据拷贝，我们需要一个支持收集操作的网络接口。这仅仅意味着等待传输的数据不需要在内存中；它可以分散在各种存储位置。在内核 2.4 版本中，修改了 `socket` 缓冲区描述符以适应这些要求 - 在 Linux 下称为零拷贝。这种方法不仅减少了多个上下文切换，还避免了处理器完成的数据拷贝。对于用户的程序不用做什么修改，所以代码仍然如下所示：

```c
sendfile(socket, file, len);
```

为了更好地了解所涉及的过程，请查看下图

![sendfile代替读写]({{ "/public/images/2019/01/sys_call_sendfile.jpg" | prepend: site.cdnurl }} "sendfile 代替读写")

1. `sendfile` 调用会导致文件内容通过 DMA 模块拷贝到内核缓冲区。
2. 没有数据被复制到 `socket` 缓冲区。相反，只有关于数据的位置和长度信息的描述符被附加到 `socket` 缓冲区。DMA 模块将数据直接从内核缓冲区传递到协议引擎，从而避免了剩余的最终拷贝。

因为数据实际上仍然是从磁盘复制到内存，从内存复制到总线，所以有人可能会认为这不是真正的零拷贝。但从操作系统的角度来看，这是零拷贝，因为内核缓冲区之间的数据不会产生多余的拷贝。使用零拷贝时，除了避免拷贝外，还可以获得其他性能优势，比如更少的上下文切换，更少的 CPU 高速缓存污染以及不会产生 CPU 校验和计算。

现在我们知道了什么是零拷贝，把前面的理论通过编码来实践。你可以从 [http://www.xalien.org/articles/source/sfl-src.tgz](http://www.xalien.org/articles/source/sfl-src.tgz) 下载源码。解压源码需要执行 `tar -zxvf sfl-src.tgz`，然后编译代码并创建一个随机数据文件 `data.bin`，接下来使用 `make` 运行。

查看头文件：

```c
/* sfl.c sendfile example program
Dragan Stancevic <
header name                 function / variable
-------------------------------------------------*/
#include <stdio.h>          /* printf, perror */
#include <fcntl.h>          /* open */
#include <unistd.h>         /* close */
#include <errno.h>          /* errno */
#include <string.h>         /* memset */
#include <sys/socket.h>     /* socket */
#include <netinet/in.h>     /* sockaddr_in */
#include <sys/sendfile.h>   /* sendfile */
#include <arpa/inet.h>      /* inet_addr */
#define BUFF_SIZE (10*1024) /* size of the tmp buffer */
```

除了 `socket` 操作需要的头文件 `<sys/socket.h>` 和 `<netinet/in.h>` 之外，我们还需要 `sendfile` 调用的头文件 - `<sys/sendfile.h>` ：

```c
/* are we sending or receiving */
if(argv[1][0] == 's') is_server++;
/* open descriptors */
sd = socket(PF_INET, SOCK_STREAM, 0);
if(is_server) fd = open("data.bin", O_RDONLY);
```

同样的程序既可以充当 *服务端/发送者*，也可以充当 *客户端/接受者*。这里我们接收一个命令提示符参数，通过该参数将标志 `is_server` 设置为以 **发送方模式** 运行。我们还打开了 `INET` 协议族的流套接字。作为在服务端运行的一部分，我们需要某种类型的数据传输到客户端，所以打开我们的数据文件（data.bin）。由于我们使用 `sendfile` 来传输数据，所以不用读取文件的实际内容将其存储在程序的缓冲区中。这是服务端地址：

```c
/* clear the memory */
memset(&sa, 0, sizeof(struct sockaddr_in));
/* initialize structure */
sa.sin_family = PF_INET;
sa.sin_port = htons(1033);
sa.sin_addr.s_addr = inet_addr(argv[2]);
```

我们重置了服务端地址结构并分配了端口和 IP 地址。服务端的地址作为命令行参数传递，端口号写死为 `1033`，选择这个端口号是因为它是一个允许访问的端口范围。

下面是服务端执行的代码分支：

```c
if(is_server){
    int client; /* new client socket */
    printf("Server binding to [%s]\n", argv[2]);
    if(bind(sd, (struct sockaddr *)&sa,
                      sizeof(sa)) < 0){
        perror("bind");
        exit(errno);
    }
}
```

作为服务端，我们需要为 `socket` 描述符分配一个地址。这是通过系统调用 `bind` 实现的，它为 `socket` 描述符（sd）分配一个服务器地址（sa）：

```c
if(listen(sd,1) < 0){
    perror("listen");
    exit(errno);
}
```

因为我们正在使用流套接字，所以我们必须接受传入连接并设置连接队列大小。我将缓冲压队列设置为 1，但对于等待接受的已建立连接，一般会将缓冲值要设置的更高一些。在旧版本的内核中，缓冲队列用于防止 `syn flood` 攻击。由于系统调用 `listen` 已经修改为 *仅为已建立的连接设置参数*，所以不使用这个调用的缓冲队列功能。内核参数 `tcp_max_syn_backlog` 代替了保护系统免受 `syn flood` 攻击的角色：

```c
if((client = accept(sd, NULL, NULL)) < 0){
    perror("accept");
    exit(errno);
}
```

`accept` 调用从挂起连接队列上的第一个连接请求创建一个新的 `socket` 连接。调用的返回值是新创建的连接的描述符； `socket` 现在可以进行读、写或轮询/select 了：

```c
if((cnt = sendfile(client,fd,&off,
                          BUFF_SIZE)) < 0){
    perror("sendfile");
    exit(errno);
}
printf("Server sent %d bytes.\n", cnt);
close(client);
```

在客户端 `socket` 描述符上建立连接，我们可以开始将数据传输到远端。通过 `sendfile` 调用来实现，该调用是在 Linux 下通过以下方式原型化的：

```c
extern ssize_t
sendfile (int __out_fd, int __in_fd, off_t *offset,
          size_t __count) __THROW;
```

- 前两个参数是文件描述符。
- 第 3 个参数指向 `sendfile` 开始发送数据的偏移量。
- 第四个参数是我们要传输的字节数。

为了使 `sendfile` 传输使用零拷贝功能，你需要从网卡获得内存收集操作支持。还需要实现校验和的协议的校验和功能，通过 TCP 或 UDP。如果你的 `NIC` 已过时不支持这些功能，你也可以使用 `sendfile` 来传输文件，不同之处在于内核会在传输之前合并缓冲区。

# 移植性问题

通常，`sendfile` 系统调用的一个问题是缺少标准实现，就像开放系统调用一样。Linux、Solaris 或 HP-UX 中 的 Sendfile 实现完全不同。这对于想通过代码实现零拷贝的开发人员而言是个问题。

其中一个实现差异是 Linux 提供了一个 `sendfile` 接口，用于在两个文件描述符（文件到文件）和（文件到socket）之间传输数据。另一方面，HP-UX 和 Solaris 只能用于文件到 socket 的提交。

第二个区别是 Linux 没有实现向量传输。Solaris sendfile 和 HP-UX sendfile 有一些扩展参数，可以避免与正在传输的数据添加头部的开销。

# 展望

Linux 下的零拷贝实现离最终实现还有点距离，并且很可能在不久的将来发生变化。要添加更多功能，例如，sendfile 调用不支持向量传输，而 Samba 和 Apache 等服务器必须使用设置了 `TCP_CORK` 标志的多个sendfile 调用。这个标志告诉系统在下一个 `sendfile` 调用中会有更多数据通过。`TCP_CORK` 和`TCP_NODELAY` 不兼容，并且在我们想要在数据前添加或附加标头时使用。这是一个完美的例子，其中向量调用将消除对当前实现所强制的多个 `sendfile` 调用和延迟的需要。

当前 sendfile 中一个相当令人不快的限制是它在传输大于2GB的文件时无法使用。如此大小的文件在今天并不罕见，并且在出路时复制所有数据相当令人失望。因为在这种情况下sendfile和mmap方法都不可用，所以sendfile64在未来的内核版本中会非常方便。

# 总结

尽管有一些缺点，不过通过 `sendfile` 来实现零拷贝也很有用，我希望你在阅读本文后可以开始在你的程序中使用它。如果想对这个主题有更深入的兴趣，请留意我的第二篇文章，标题为 “零拷贝 - 内核态分析”，我将在零拷贝的内核内部挖掘更多内容。

> 英文原文：[http://www.linuxjournal.com/article/6345](http://www.linuxjournal.com/article/6345)