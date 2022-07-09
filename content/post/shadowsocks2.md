---
title: Shadowsocks (二) TCPRelayHandler
date: 2022-07-10T02:23:58+08:00
draft: true
---

这一次我们主要分析一下 `TCPRelayHandler` 这个类, 这个类如前文所述, 是 `TCPRelay` 创建出来, 处理单个 TCP 连接的一个类. 其主要通过 handle_event 函数来处理 IO 事件.

## 遗留问题

回顾一下我们之前的遗留问题:

1. daemon 进程是怎么启动的呢? 为什么要等最后 `loop.run()` 之前再 `set_uesr` 呢?
2. `loop` 是怎么工作的?
   1. 为什么还需要判断 handler 不为空, 什么情况下它会是空呢?
   2. 最后看起来是在判断超时的几行代码的意义是什么呢?
3. 三个 server 分别是怎么工作的呢?
   1. 为什么 TCPRelay 生成完新 handler 不直接将对应的 socket 处理函数注册成新生成的 handler, 反而要把它注册成生成了这个 handler 的TCPRelay 呢?

这里详细分析 `TCPRelayHandler` 主要是希望通过详细阅读其源码来尝试回答 3.1 以及简单了解一下处理 TCP 连接的方式.

首先, 为什么说 3.1 这个问题中的设计是很蠢的呢? 因为这意味着我们额外维护了一个 TCPRelay 中的哈希表, 这导致一个老连接的套接字发生IO事件的时候, 我们需要首先用这个 `fd` 在 loop 的 `_fdmap` 中找到这个 `TCPRelay` 然后再在 `TCPRelay` 的 `_fd_to_handlers` 中再做一次哈希, 用用一个 `fd` 再次找到对应的 `TCPRelayHandler`, 如果统一两个类的 handler 的 handle_event 接口统一, 然后直接将 `TCPRelayHandler` 注册到 loop 的 `_fdmap` 中就可以省一次哈希.

首先说一个疑点:

在[这里](https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L865), 我们可以看到一个 warming, 说我们 epoll 出来一个被 removed 的 `fd` 这个事情是需要被警告的, 而我目前还不知道什么情况下才会出现这个情况. 完全可能是由于这个地方, 导致了采用了这种看起来低效的设计方式.

好我们接下来开始分析 `TCPRelayHandler`.

## `handle_event` 

我们在[这里](https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L656)可以看到 `TCPRelayHandler` 的 `handle_event` 函数:

```python
    def handle_event(self, sock, event):
        # handle all events in this handler and dispatch them to methods
        if self._stage == STAGE_DESTROYED:
            logging.debug('ignore handle_event: destroyed')
            return
        # order is important
        if sock == self._remote_sock:
            if event & eventloop.POLL_ERR:
                self._on_remote_error()
                if self._stage == STAGE_DESTROYED:
                    return
            if event & (eventloop.POLL_IN | eventloop.POLL_HUP):
                self._on_remote_read()
                if self._stage == STAGE_DESTROYED:
                    return
            if event & eventloop.POLL_OUT:
                self._on_remote_write()
        elif sock == self._local_sock:
            if event & eventloop.POLL_ERR:
                self._on_local_error()
                if self._stage == STAGE_DESTROYED:
                    return
            if event & (eventloop.POLL_IN | eventloop.POLL_HUP):
                self._on_local_read()
                if self._stage == STAGE_DESTROYED:
                    return
            if event & eventloop.POLL_OUT:
                self._on_local_write()
        else:
            logging.warn('unknown socket')
```

而我们实际上可以在[这里](https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L117)看到:
```python
        self._local_sock = local_sock
        self._remote_sock = None
```
所以实际上这里我们可以知道上面那个 `handle_event` 函数实际上在 sslocal 时运行的是下面的那个 `elif`, 也就是说实际上重要的是以下三个函数:

1.  self._on_local_error()
2.  self._on_local_read()
3.  self._on_local_write()

我们以下一个一个看这些函数.

## `_on_local_error`

这个[函数](https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L643)相当的简短, 也非常好懂, 

这里就不多废话了, 重点肯定是下面两个函数.

在看之前我们首先确定一下 epoll 中提及的几个 socket 发生的 IO 事件到底意味着什么.

## epoll

从[这里](https://docs.python.org/3.8/library/select.html?highlight=epoll#epoll-objects)可以了解到 epoll 中拉出来的几个 event 都是什么意思

| Constant | Meaning |
| :--- | ---: |
| EPOLLIN | Available for read |
| EPOLLOUT | Available for write |
| EPOLLPRI | Urgent data for read |
| EPOLLERR | Error condition happened on the assoc. fd |
| EPOLLHUP | Hang up happened on the assoc. fd |
| EPOLLET | Set Edge Trigger behavior, the default is Level Trigger behavior |
| EPOLLONESHOT | Set one-shot behavior. After one event is pulled out, the fd is internally disabled |
| EPOLLEXCLUSIVE | Wake only one epoll object when the associated fd has an event. The default (if this flag is not set) is to wake all epoll objects polling on a fd. |
| EPOLLRDHUP | Stream socket peer closed connection or shut down writing half of connection. |
| EPOLLRDNORM | Equivalent to EPOLLIN |
| EPOLLRDBAND | Priority data band can be read. |
| EPOLLWRNORM | Equivalent to EPOLLOUT |
| EPOLLWRBAND | Priority data may be written. |
| EPOLLMSG | Ignored. |

上面有出现的应该是 EPOLLIN EPOLLHUP EPOLLOUT EPOLLERR 这四个.

也就是这四个:

| Constant | Meaning |
| :--- | ---: |
| EPOLLIN | Available for read |
| EPOLLOUT | Available for write |
| EPOLLERR | Error condition happened on the assoc. fd |
| EPOLLHUP | Hang up happened on the assoc. fd | 

## `_on_local_read`

当事件 EPOLLIN 发生, 也就是 socket 中有数据可以读了的时候, 或者 EPOLLHUP 事件发生, 也就是这个 socket 被挂断的时候, 会触发这个函数.

根据 [这里](https://stackoverflow.com/questions/52976152/tcp-when-is-epollhup-generated) 的说法. EPOLLHUP 这个信号只在 FIN 包被交换之后产生. 也就是整个 TCP 连接完全关闭的时候.

[代码](https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L552) 也很短, 就直接粘贴过来吧:

```python
    def _on_local_read(self):
        # handle all local read events and dispatch them to methods for
        # each stage
        if not self._local_sock:
            return
        is_local = self._is_local
        data = None
        if is_local:
            buf_size = UP_STREAM_BUF_SIZE
        else:
            buf_size = DOWN_STREAM_BUF_SIZE
        try:
            data = self._local_sock.recv(buf_size)
        except (OSError, IOError) as e:
            if eventloop.errno_from_exception(e) in \
                    (errno.ETIMEDOUT, errno.EAGAIN, errno.EWOULDBLOCK):
                return
        if not data:
            self.destroy()
            return
        self._update_activity(len(data))
        if not is_local:
            data = self._cryptor.decrypt(data)
            if not data:
                return
        if self._stage == STAGE_STREAM:
            self._handle_stage_stream(data)
            return
        elif is_local and self._stage == STAGE_INIT:
            # jump over socks5 init
            if self._is_tunnel:
                self._handle_stage_addr(data)
                return
            else:
                self._handle_stage_init(data)
        elif self._stage == STAGE_CONNECTING:
            self._handle_stage_connecting(data)
        elif (is_local and self._stage == STAGE_ADDR) or \
                (not is_local and self._stage == STAGE_INIT):
            self._handle_stage_addr(data)
```

在 [这里](https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L47) 可以看到每个 stage 的具体含意.

```python
# for each opening port, we have a TCP Relay

# for each connection, we have a TCP Relay Handler to handle the connection

# for each handler, we have 2 sockets:
#    local:   connected to the client
#    remote:  connected to remote server

# for each handler, it could be at one of several stages:

# as sslocal:
# stage 0 auth METHOD received from local, reply with selection message
# stage 1 addr received from local, query DNS for remote
# stage 2 UDP assoc
# stage 3 DNS resolved, connect to remote
# stage 4 still connecting, more data from local received
# stage 5 remote connected, piping local and remote

# as ssserver:
# stage 0 just jump to stage 1
# stage 1 addr received from local, query DNS for remote
# stage 3 DNS resolved, connect to remote
# stage 4 still connecting, more data from local received
# stage 5 remote connected, piping local and remote

STAGE_INIT = 0
STAGE_ADDR = 1
STAGE_UDP_ASSOC = 2
STAGE_DNS = 3
STAGE_CONNECTING = 4
STAGE_STREAM = 5
STAGE_DESTROYED = -1
```

那基本就可以知道这东西的工作方式了.

回顾一下我们之前的遗留问题:

1. daemon 进程是怎么启动的呢? 为什么要等最后 `loop.run()` 之前再 `set_uesr` 呢?
2. `loop` 是怎么工作的?
   1. 为什么还需要判断 handler 不为空, 什么情况下它会是空呢?
   2. 最后看起来是在判断超时的几行代码的意义是什么呢?
3. 三个 server 分别是怎么工作的呢?
   1. 为什么 TCPRelay 生成完新 handler 不直接将对应的 socket 处理函数注册成新生成的 handler, 反而要把它注册成生成了这个 handler 的TCPRelay 呢?

这里说一下 2.1

我们来看下 self.destroy(), 在[这里](https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L718) 可以看到, 我们从 server 的 map 中 remove 了 这个 handler, 

看了一下 destory 我感觉 loop 中的 handler 根本不可能是空的. 因为 destory 的时候实际上把所有和这个 socket 相关的资源都释放了, 当然也将 socket 从 epoll 的监听列表中去掉了.

所以我们不太可能在 epoll 反回的列表中发现一个 handler 已经被 remove 的 socket.

感觉这个设计相当奇怪, 有很多无用的检查代码. 不过可能这样也挺好的. 不太容易出错.

3.1 仍然没有得到回答, 感觉这么设计就是低效的.

## `_on_local_write`

```python
    def _on_local_write(self):
        # handle local writable event
        if self._data_to_write_to_local:
            data = b''.join(self._data_to_write_to_local)
            self._data_to_write_to_local = []
            self._write_to_sock(data, self._local_sock)
        else:
            self._update_stream(STREAM_DOWN, WAIT_STATUS_READING)
```

这个函数相当直白

# 下一步

这个 python 写的 ss 看起来设计上有一定问题, 首先完全是单线程的, 其次如这两篇博客所述, 总体设计上貌似存在一些问题.

我们接下来看看 go 写的 [go-shadowsocks2](https://github.com/shadowsocks/go-shadowsocks2) 吧.

