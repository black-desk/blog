---
title: Shadowsocks (一) 主循环
date: 2022-07-10T02:23:58+08:00
draft: false
---
这里简单分析一下Shadowsocks这个软件的源码.

因为网上一直有声音认为其源码质量不错.刚好最近在学一些服务器编程相关的知识,所以就
来简单分析一下它.

同时由于希望能获得一些阅读源码的经验. 这里会简单记录我读源码的心路历程.

## 目录结构

首先我们来tree一下,惯例,测试相关的文件就省略了

    ❯ tree
    .
    ├── CHANGES
    ├── CONTRIBUTING.md
    ├── Dockerfile
    ├── LICENSE
    ├── MANIFEST.in
    ├── README.md
    ├── README.rst
    ├── config.json.example
    ├── debian
    │   ├── changelog
    │   ├── compat
    │   ├── config.json
    │   ├── control
    │   ├── copyright
    │   ├── docs
    │   ├── init.d
    │   ├── install
    │   ├── rules
    │   ├── shadowsocks.default
    │   ├── shadowsocks.manpages
    │   ├── source
    │   │   └── format
    │   ├── sslocal.1
    │   └── ssserver.1
    ├── setup.py
    ├── shadowsocks
    │   ├── __init__.py
    │   ├── asyncdns.py
    │   ├── common.py
    │   ├── crypto
    │   │   ├── __init__.py
    │   │   ├── aead.py
    │   │   ├── hkdf.py
    │   │   ├── mbedtls.py
    │   │   ├── openssl.py
    │   │   ├── rc4_md5.py
    │   │   ├── sodium.py
    │   │   ├── table.py
    │   │   └── util.py
    │   ├── cryptor.py
    │   ├── daemon.py
    │   ├── eventloop.py
    │   ├── local.py
    │   ├── lru_cache.py
    │   ├── manager.py
    │   ├── server.py
    │   ├── shell.py
    │   ├── tcprelay.py
    │   ├── tunnel.py
    │   └── udprelay.py
    ├── snapcraft.yaml
    ├── tests
    │   └── ... 
    └── utils
        ├── README.md
        ├── autoban.py
        └── fail2ban
            └── shadowsocks.conf
    
    11 directories, 106 files

可以看到,主要的源码应该都在shadowsocks目录下,我们可以看到用于加/解密的程序,可以
看到处理TCP,UDP连接的程序.

也可以看到根目录下有DockerFile,以及一个不知道用来干啥的debain文件夹,我猜测是用来
生成软件包的.

setup.py看起来像是用于安装成Python包的.

现在对于目录结构有一定认识和猜测了之后,我们可以开始下一步行动了.

## 入口

阅读源码肯定要先找入口,至少要知道`main`在哪里,

由于实际上使用过这个软件,我知道我们在本地启动的时候输入的命令是`sslocal`,而在服
务器运行的时候输入的命令是`ssserver`.

这意味着我们首先得知道这两个命令是如何启动代码的,输入这两个命令之后实际上运行了
什么函数.

在[`setup.py`][==link1==]中我们可以看到:

``` python
entry_points="""
[console_scripts]
sslocal = shadowsocks.local:main
ssserver = shadowsocks.server:main
""",
```

应该不难看出,上面的代码意味着:`sslocal`对应的是`shadowsocks`包下的`local.py`里面
的`main()`,而`ssserver`对应的是`shadowsocks`包下的`server.py`里面的`main()`.

那么我们先看客户端,也就是`sslocal`.

## 客户端 sslocal

### `main`

[`local.py::main`][==link2==]

这个函数不长,我就直接复制过来了.

``` python
@shell.exception_handle(self_=False, exit_code=1)
def main():
    shell.check_python()

    # fix py2exe
    if hasattr(sys, "frozen") and sys.frozen in \
            ("windows_exe", "console_exe"):
        p = os.path.dirname(os.path.abspath(sys.executable))
        os.chdir(p)

    config = shell.get_config(True) # shell大约是一个"外壳",用来交互.所以我们使用shell来获得配置文件.
    daemon.daemon_exec(config) # 启动daemon进程

    logging.info("starting local at %s:%d" %
                 (config['local_address'], config['local_port'])) # 可以看出来监听的地址和端口是写在配置文件里的.

    dns_resolver = asyncdns.DNSResolver() # 启动dns,这个dns貌似是不可配置的,因为config没有被传入. 
    tcp_server = tcprelay.TCPRelay(config, dns_resolver, True) # 创建新的tcp_server和udp_server
    udp_server = udprelay.UDPRelay(config, dns_resolver, True)
    loop = eventloop.EventLoop()
    dns_resolver.add_to_loop(loop)
    tcp_server.add_to_loop(loop)
    udp_server.add_to_loop(loop) # 建立一个loop将dns/udp/tcp server都添加到这个loop中.

    def handler(signum, _):
        logging.warn('received SIGQUIT, doing graceful shutting down..')
        tcp_server.close(next_tick=True)
        udp_server.close(next_tick=True)
    signal.signal(getattr(signal, 'SIGQUIT', signal.SIGTERM), handler) # 注册信号处理函数.

    def int_handler(signum, _):
        sys.exit(1)
    signal.signal(signal.SIGINT, int_handler) # 注册信号处理函数.

    daemon.set_user(config.get('user', None))
    loop.run() # 启动
```

[这里][==link3==]有关于两个信号`SIGQUIT`和`SIGINT`的区别.

这里可以记下这么几个问题:

1. daemon进程是怎么启动的呢?为什么要等最后`loop.run()`之前再`set_user`呢?
2. 为什么dns服务器创建时不读取配置文件?
3. `loop`是怎么工作的?
4. 三个`..._server`分别是怎么工作的呢?

可以很明显地看出,我们的主要目标应该是3和4.

那么接下来的目标就是`loop.run()`了.

### `loop.run`

看一个类的时候我们可以先看它是如何工作的.有什么看不懂的地方再回头看它的构造函数
之类的东西.

对于一个过程其实也是一样的.我们可以先看它是如何工作的,然后再看它是怎么初始化的.

落实到这里,就是说我们可以先看`run()`看不懂了再去看`add_to_loop()`

看看[`eventloop.py:run()`][==link4==]

``` python
def run(self):
    events = []
    while not self._stopping:
        asap = False
        try:
            events = self.poll(TIMEOUT_PRECISION) # poll 出一个 events, 从下面看 events 中有 sock, fd 和 event 这三个东西.
        except (OSError, IOError) as e:
            if errno_from_exception(e) in (errno.EPIPE, errno.EINTR):
                # EPIPE: Happens when the client closes the connection
                # EINTR: Happens when received a signal
                # handles them as soon as possible
                asap = True
                logging.debug('poll:%s', e)
            else:
                logging.error('poll:%s', e)
                traceback.print_exc()
                continue

        for sock, fd, event in events:
            handler = self._fdmap.get(fd, None) # 获得和 fd 对应的 handler
            if handler is not None:
                handler = handler[1]
                try:
                    handler.handle_event(sock, fd, event) # 调用 handler 处理 IO 事件.
                except (OSError, IOError) as e:
                    shell.print_exception(e)
        now = time.time()
        if asap or now - self._last_time >= TIMEOUT_PRECISION:
            for callback in self._periodic_callbacks:
                callback()
            self._last_time = now
```

大致上这个loop的工作流程就是:

poll出来一个IO事件,看下这个socket有没有对应的handler,有就去调这个handler.

这里扩充一下问题列表:

1. daemon进程是怎么启动的呢?为什么要等最后`loop.run()`之前再`set_uesr`呢?
2. 为什么dns服务器创建时不读取配置文件?
3. `loop`是怎么工作的?
   1. handler这个对象是如何注册到loop中的?\[是在什么地方被放进`_fdmap`的呢?\]
   
   2. `handler[1]`?
   
   3. 为什么还需要判断`handler`不为空,什么情况下它会是空呢?
   
   4. 为什么`events`遍历的时候会出来`sock`,`fd`和`event`三个变量呢?`epoll`不是只
      会有`fd`和`event`两个东西告诉程序是哪个文件发生了什么样的事件吗?
   
   5. 最后看起来是在判断超时的几行代码的意义是什么呢?
4. 三个server分别是怎么工作的呢?

那么重点应该是3.1,我们可以猜测这个注册发生在`add_to_loop`里面.但是我们首先可以发
现loop有个方法叫add,这里写了`_fdmap`.

[`eventloop.py:add()`][==link5==]

``` python
def add(self, f, mode, handler):
    fd = f.fileno()
    self._fdmap[fd] = (f, handler)
    self._impl.register(fd, mode)
```

这里有一个注册函数.

在[`eventloop.py::__init__()`][==link6==]中可以看到:我们的`_impl`就
是`select.epoll`.

那么我们现在可以看看是谁在调用这个`add`,以`TCPRelay`的`add_to_loop`为例:

### `TCPRelay`

在[这里][==link7==]可以看到

``` python
def add_to_loop(self, loop):
    if self._eventloop:
        raise Exception('already add to loop')
    if self._closed:
        raise Exception('already closed')
    self._eventloop = loop
    self._eventloop.add(self._server_socket,
                        eventloop.POLL_IN | eventloop.POLL_ERR, self)
    self._eventloop.add_periodic(self.handle_periodic)
```

这里将自己的`_server_socket`注册给了`loop`,这个`socket`肯定是创建`TCPRelay`的时
候建立的.

我们来看看这个创建过程:

在[这里][==link8==] 我们可以看到:

``` python
def __init__(self, config, dns_resolver, is_local, stat_callback=None):
    self._config = config
    self._is_local = is_local
    self._dns_resolver = dns_resolver
    self._closed = False
    self._eventloop = None
    self._fd_to_handlers = {}
    self._is_tunnel = False

    self._timeout = config['timeout']
    self._timeouts = []  # a list for all the handlers
    # we trim the timeouts once a while
    self._timeout_offset = 0   # last checked position for timeout
    self._handler_to_timeouts = {}  # key: handler value: index in timeouts

    if is_local:
        listen_addr = config['local_address']
        listen_port = config['local_port']
    else:
        listen_addr = config['server']
        listen_port = config['server_port']
    self._listen_port = listen_port

    addrs = socket.getaddrinfo(listen_addr, listen_port, 0,
                               socket.SOCK_STREAM, socket.SOL_TCP)
    if len(addrs) == 0:
        raise Exception("can't get addrinfo for %s:%d" %
                        (listen_addr, listen_port))
    af, socktype, proto, canonname, sa = addrs[0]
    server_socket = socket.socket(af, socktype, proto)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind(sa)
    server_socket.setblocking(False)
    if config['fast_open']:
        try:
            server_socket.setsockopt(socket.SOL_TCP, 23, 5)
        except socket.error:
            logging.error('warning: fast open is not available')
            self._config['fast_open'] = False
    server_socket.listen(1024)
    self._server_socket = server_socket
    self._stat_callback = stat_callback
```

这就是创建socket,bind,listen的三部曲了.这里还能看到一个Fast Open的设置.

使用了非阻塞的 socket.

接下来的问题就是:

``` python
self._eventloop.add(self._server_socket,
                        eventloop.POLL_IN | eventloop.POLL_ERR, self)
```

这个地方,向`_eventloop`注册事件的时候,实际上添加进去的handle是self.

在上面这里的[`eventloop.py:run()`][==link4==]中:

``` python
...
        for sock, fd, event in events:
            handler = self._fdmap.get(fd, None) # 获得和fd对应的handler
            if handler is not None:
                handler = handler[1]
                try:
                    handler.handle_event(sock, fd, event) # 调用handler处理IO事件.
                except (OSError, IOError) as e:
                    shell.print_exception(e)
...
```

调用了`handle_event()`方法,那我们来看一下

[这里][==link9==]

``` python
    def handle_event(self, sock, fd, event):
        # handle events and dispatch to handlers
        if sock:
            logging.log(shell.VERBOSE_LEVEL, 'fd %d %s', fd,
                        eventloop.EVENT_NAMES.get(event, event))
        if sock == self._server_socket: # 如果是监听新连接的socket
            if event & eventloop.POLL_ERR:
                # TODO
                raise Exception('server_socket error')
            try:
                logging.debug('accept')
                conn = self._server_socket.accept()
                TCPRelayHandler(self, self._fd_to_handlers, # 创建处理单个连接的handler,并将其注册到eventloop中.
                                self._eventloop, conn[0], self._config,
                                self._dns_resolver, self._is_local)
            except (OSError, IOError) as e:
                error_no = eventloop.errno_from_exception(e)
                if error_no in (errno.EAGAIN, errno.EINPROGRESS,
                                errno.EWOULDBLOCK):
                    return
                else:
                    shell.print_exception(e)
                    if self._config['verbose']:
                        traceback.print_exc()
        else: 
            if sock: # 如果这个socket不是监听新连接的.那么这意味着我们可以在一个map里面找到它的handler.调用它.
                handler = self._fd_to_handlers.get(fd, None)
                if handler:
                    handler.handle_event(sock, event)
            else:
                logging.warn('poll removed fd')
```

可以在`TCPRelayHandler`的[构造函数][==link10==]中看到:

``` python
 def __init__(self, server, fd_to_handlers, loop, local_sock, config,
              dns_resolver, is_local):
        self._server = server
        ...
        loop.add(local_sock, eventloop.POLL_IN | eventloop.POLL_ERR,
                 self._server)
        ...
```

这里产生了一个新的疑问,为什么创建好新连接进来的socket对应的handler之后
向eventloop中注册处理函数的时候不干脆直接注册成新生成的handler呢?还将它注册
成`TCPRelay`呢?

更新一下疑问列表:

1. daemon进程是怎么启动的呢?为什么要等最后`loop.run()`之前再`set_uesr`呢?
2. 为什么dns服务器创建时不读取配置文件?
3. `loop`是怎么工作的?
   1. handler这个对象是如何注册到loop中的?\[是在什么地方被放进`_fdmap`的呢?\]
   
   2. `handler[1]`?
   
   3. 为什么还需要判断handler不为空,什么情况下它会是空呢?
   
   4. 为什么`events`遍历的时候会出来`sock` `fd`和`event`三个变量呢?`epoll`不是只
      会有`fd`和`event`两个东西告诉程序是哪个文件发生了什么样的事件吗?
   
   5. 最后看起来是在判断超时的几行代码的意义是什么呢?
4. 三个server分别是怎么工作的呢?
   1. 为什么TCPRelay生成完新handler不直接将对应的socket处理函数注册成新生成的
      handler,反而要把它注册成生成了这个handler的TCPRelay呢?

不过总体来说整个流程就十分清楚了.

## 主要流程

我们有一个主循环,这个主事件循环从epoll中拉出发生了IO事件的`fd`,然后从一个map中找
到这个`fd`对应的handler,调用这个handler的`handle_event`函数来处理这个`fd`发生
的IO事件.

对于TCP连接的处理而言,我们把所有TCP连接的handle注册成了TCPRelay.也就是说所有TCP
接对应的socket发生IO事件之后,都会调用TCPRelay下面的`handle_event`.如果是老连接,
我们就从TCPRelay下的另一个`fd`到handle的map里面找到这个socket对应的handle,调用
handle来处理IO事件;如果是新连接,就创建一个新的`TCPRelayHandler`,将它写进TCPRelay
下的map.

大致就是这样了.

至于`TCPRelaHandler`是怎么处理IO事件的.等我想看了,再写一篇文章就是了.

## 问题

那么我们现在把之前遗留的几个问题一一处理一下:

1. daemon进程是怎么启动的呢?为什么要等最后`loop.run()`之前再`set_user`呢?

2. 为什么dns服务器创建时不读取配置文件?
   
   这里只是在本地端创建dns不需要读取配置文件.sever端的[代码
   中](https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/server.py#L61)
   可以看到:
   
   ``` python
   dns_resolver = asyncdns.DNSResolver(config['dns_server'],
                                       config['prefer_ipv6'])
   ```

3. `loop`是怎么工作的?
   
   1. handler这个对象是如何注册到loop中的?\[是在什么地方被放进`_fdmap`的呢?\]
      
      回答过了
   
   2. `handler[1]`?
      
      在[这里][==link11==]可以看到
      
      ``` python
      self._fdmap = {}  # (f, handler)
      ```
      
      这里的`f`是那个file对象.所以这里需要取第二个,第二个才是真正的handler.
   
   3. 为什么还需要判断handler不为空,什么情况下它会是空呢?
   
   4. 为什么`events`遍历的时候会出来`sock` `fd`和`event`三个变量呢?`epoll`不是只
      会有`fd`和`event`两个东西告诉程序是哪个文件发生了什么样的事件吗?
      
      [这里][==link12==]可以看到,我们通过这个`_fdmap`取到了`fd`对应的file对象.
      
      ``` python
      return [(self._fdmap[fd][0], fd, event) for fd, event in events]
      ```
   
   5. 最后看起来是在判断超时的几行代码的意义是什么呢?

4. 三个server分别是怎么工作的呢?
   
   1. 为什么TCPRelay生成完新handler不直接将对应的socket处理函数注册成新生成
      的handler,反而要把它注册成生成了这个handler的TCPRelay呢?

## 遗留问题

那么现在留下的问题是这几个:

1. daemon进程是怎么启动的呢?为什么要等最后`loop.run()`之前再`set_user`呢?

2. `loop`是怎么工作的?
   
   1. 为什么还需要判断handler不为空,什么情况下它会是空呢?
   
   2. 最后看起来是在判断超时的几行代码的意义是什么呢?

3. 三个server分别是怎么工作的呢?
   
   1. 为什么TCPRelay生成完新handler不直接将对应的socket处理函数注册成新生成
      的handler,反而要把它注册成生成了这个handler的TCPRelay呢?

以及我们接下来还有一些希望了解到的内容,比如`TCPRelayHandler`到底需要处理哪些IO事
件,分别又都是怎么处理的呢?

[==link1==]: https://github.com/shadowsocks/shadowsocks/blob/master/setup.py#L21
[==link2==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/local.py#L30
[==link3==]: https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html
[==link4==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/eventloop.py#L193
[==link5==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/eventloop.py#L170
[==link6==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/eventloop.py#L147
[==link7==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L766
[==link8==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L723
[==link9==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L835
[==link10==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/tcprelay.py#L152
[==link11==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/eventloop.py#L160
[==link12==]: https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/eventloop.py#L168
