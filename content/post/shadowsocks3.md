---
title: go-shadowsocks2 源码简析
date: 2022-07-10T02:23:58+08:00
draft: true
---

我们现在来简单分析一下 go-shadowsocks2 这个项目.

第一篇就简单分析一下这个项目大体的处理流程.

go 语言的项目找入口就相当好找了. 一眼就可以看到 main 函数在哪了.

来看看吧.

照例, 先看 client

在 [main](https://github.com/shadowsocks/go-shadowsocks2/blob/master/main.go#L27) 函数中可以看到:

```go
//...
func main() {
//...
	flag.StringVar(&flags.Client, "c", "", "client connect address or url")
  flag.StringVar(&flags.Socks, "socks", "", "(client-only) SOCKS listen address")
//...
	if flags.Client != "" { // client mode
		addr := flags.Client
		cipher := flags.Cipher
		password := flags.Password
		var err error
//...
		ciph, err := core.PickCipher(cipher, key, password)
		if err != nil {
			log.Fatal(err)
		}
//...
		if flags.Socks != "" {
//...
			go socksLocal(flags.Socks, addr, ciph.StreamConn)
//...
    }
//...
  }
//...
```

可以看到 [socksLocal](https://github.com/shadowsocks/go-shadowsocks2/blob/master/tcp.go#L17) 这个函数就是用来监听, 接受新连接的了.

先不急着看它的源码, 我们可以先看看我们到底传了什么给它.

主要就是这个 ciph.StreamConn 到底是啥.

它是从 [core.PickCipher](https://github.com/shadowsocks/go-shadowsocks2/blob/master/tcp.go#L17) 来的.

```go
//...
// PickCipher returns a Cipher of the given name. Derive key from password if given key is empty.
func PickCipher(name string, key []byte, password string) (Cipher, error) {
	name = strings.ToUpper(name)

	switch name {
	case "DUMMY":
		return &dummy{}, nil
	case "CHACHA20-IETF-POLY1305":
		name = aeadChacha20Poly1305
	case "AES-128-GCM":
		name = aeadAes128Gcm
	case "AES-256-GCM":
		name = aeadAes256Gcm
	}

	if choice, ok := aeadList[name]; ok {
		if len(key) == 0 {
			key = kdf(password, choice.KeySize)
		}
		if len(key) != choice.KeySize {
			return nil, shadowaead.KeySizeError(choice.KeySize)
		}
		aead, err := choice.New(key)
		return &aeadCipher{aead}, err
	}

	return nil, ErrCipherNotSupported
}
//...
```

那么这个 Cipher.StreamConn 是什么来头呢?
在 [这里](https://github.com/shadowsocks/go-shadowsocks2/blob/master/core/cipher.go#L13) 可以看到:

```go
//...
type Cipher interface {
	StreamConnCipher
	PacketConnCipher
}

type StreamConnCipher interface {
	StreamConn(net.Conn) net.Conn
}

type PacketConnCipher interface {
	PacketConn(net.PacketConn) net.PacketConn
}
//...
```

可以看到 StreamConn 这个方法来自于 StreamConnCipher 这个接口, 我们可以在 [这个](https://github.com/shadowsocks/go-shadowsocks2/blob/master/core/cipher.go#L86) 地方找到 StreamConn 的一个实现:

```go
//...
func (aead *aeadCipher) StreamConn(c net.Conn) net.Conn { return shadowaead.NewConn(c, aead) }
//...
```

就在这段代码的下面, 我们可以看到:

```go
// dummy cipher does not encrypt
type dummy struct{}

func (dummy) StreamConn(c net.Conn) net.Conn             { return c }
func (dummy) PacketConn(c net.PacketConn) net.PacketConn { return c }
```

那么这个方法大概是建立一个加密的连接之类的?

那下面的那个 PacketConn 多半就是给 UDP 用的了吧?

我们来看看这个 shadowaead.NewConn 吧:

```go
//...
// NewConn wraps a stream-oriented net.Conn with cipher.
func NewConn(c net.Conn, ciph Cipher) net.Conn { return &streamConn{Conn: c, Cipher: ciph} }
```

这里可以看到它实际上是返回了一个这样的 struct.

那么现在回到正轨, 我们继续分析 socksLocal 看看它是如何处理 tcp 连接的:

```go
//...
// Create a SOCKS server listening on addr and proxy to server.
func socksLocal(addr, server string, shadow func(net.Conn) net.Conn) {
	logf("SOCKS proxy %s <-> %s", addr, server)
	tcpLocal(addr, server, shadow, func(c net.Conn) (socks.Addr, error) { return socks.Handshake(c) })
}
//...
```

这个 [tcpLocal](https://github.com/shadowsocks/go-shadowsocks2/blob/master/tcp.go#L34) 的定义在这里:

它并不长, 而且就是整个 tcp 连接处理框架的核心了.

```go
//...
// Listen on addr and proxy to server to reach target from getAddr.
func tcpLocal(addr, server string, shadow func(net.Conn) net.Conn, getAddr func(net.Conn) (socks.Addr, error)) {
	l, err := net.Listen("tcp", addr)
	if err != nil {
		logf("failed to listen on %s: %v", addr, err)
		return
	}

	for { // 这个循环用来监听新连接
		c, err := l.Accept() // 接受新连接
		if err != nil {
			logf("failed to accept: %s", err)
			continue
		}

		go func() { // 对于每个连接, 我们开一个 goroutine 来处理.
			defer c.Close()
			tgt, err := getAddr(c)
      // ... 跳过错误处理
			
      rc, err := net.Dial("tcp", server)
      // ... 跳过错误处理

			defer rc.Close()
			if config.TCPCork {
				rc = timedCork(rc, 10*time.Millisecond, 1280)
			}
			rc = shadow(rc) // 这里实际上调用了之前说的 shadowaead.NewConn

			if _, err = rc.Write(tgt); err != nil { // 这里的 Write 函数可以是 streamConn.Write
```
它的实现可以在 [这里](https://github.com/shadowsocks/go-shadowsocks2/blob/master/shadowaead/stream.go#L257) 找到.
```go
				logf("failed to send target address: %v", err)
				return
			}

			logf("proxy %s <-> %s <-> %s", c.RemoteAddr(), server, tgt)
			if err = relay(rc, c); err != nil { // 这里的 relay 就是主要负责交换数据的函数了.
				logf("relay error: %v", err)
			}
		}()
    // 这里可以看到事实上处理完一开始的一些初始化, 得到两个 conn (一个是 c, 一个是 rc) 之后, 
    // 接下来交换数据相关的操作都是 relay 函数在做了.
	}
}
//...
```

重点肯定是 relay 函数, 它的定义在[这里](https://github.com/shadowsocks/go-shadowsocks2/blob/master/tcp.go#L145):

基本就是建立了两个 gorountine 来负责读和写数据.

```go
//...
// relay copies between left and right bidirectionally
func relay(left, right net.Conn) error {
	var err, err1 error
	var wg sync.WaitGroup
	var wait = 5 * time.Second
	wg.Add(1)
	go func() {
		defer wg.Done()
		_, err1 = io.Copy(right, left)
		right.SetReadDeadline(time.Now().Add(wait)) // unblock read on right
	}()
	_, err = io.Copy(left, right)
	left.SetReadDeadline(time.Now().Add(wait)) // unblock read on left
	wg.Wait()
	if err1 != nil && !errors.Is(err1, os.ErrDeadlineExceeded) { // requires Go 1.15+
		return err1
	}
	if err != nil && !errors.Is(err, os.ErrDeadlineExceeded) {
		return err
	}
	return nil
}
//...
```

那么基本的流程就都清楚了.

接下来稍微还有点兴趣的部份就是 **加密** 和 **解密** 的部份了.

这显然是由 streamConn.Write 和 Read 调用了 aeadCipher 中的一些东西提供的.

先看 Write 吧.

函数的定义在 [这里](https://github.com/shadowsocks/go-shadowsocks2/blob/master/shadowaead/stream.go#L257)

```go
func (c *streamConn) Write(b []byte) (int, error) {
	if c.w == nil {
		if err := c.initWriter(); err != nil {
			return 0, err
		}
	}
	return c.w.Write(b)
}
```
可以看到这个 Write 的初始化是 lazy 的.

那么肯定就要看这个 initWriter 了, 它的定义在[这里](https://github.com/shadowsocks/go-shadowsocks2/blob/master/shadowaead/stream.go#L239).

```go
func (c *streamConn) initWriter() error {
	salt := make([]byte, c.SaltSize())
	if _, err := io.ReadFull(rand.Reader, salt); err != nil {
		return err
	}
	aead, err := c.Encrypter(salt) // 加了个盐
	if err != nil {
		return err
	}
	_, err = c.Conn.Write(salt) // 盐发出去
	if err != nil {
		return err
	}
	internal.AddSalt(salt) // ???
	c.w = newWriter(c.Conn, aead) // 新建 writer
	return nil
}
```

打问号的地方我们之后再说, 我们这里先看看这个 [newWriter](https://github.com/shadowsocks/go-shadowsocks2/blob/master/shadowaead/stream.go#L26).

```go
type writer struct {
	io.Writer
	cipher.AEAD
	nonce []byte
	buf   []byte
}
//...
func newWriter(w io.Writer, aead cipher.AEAD) *writer {
	return &writer{
		Writer: w,
		AEAD:   aead,
		buf:    make([]byte, 2+aead.Overhead()+payloadSizeMask+aead.Overhead()),
		nonce:  make([]byte, aead.NonceSize()),
	}
}

// Write encrypts b and writes to the embedded io.Writer.
func (w *writer) Write(b []byte) (int, error) {
	n, err := w.ReadFrom(bytes.NewBuffer(b))
	return int(n), err
}

// ReadFrom reads from the given io.Reader until EOF or error, encrypts and
// writes to the embedded io.Writer. Returns number of bytes read from r and
// any error encountered.
func (w *writer) ReadFrom(r io.Reader) (n int64, err error) {
	for {
		buf := w.buf
		payloadBuf := buf[2+w.Overhead() : 2+w.Overhead()+payloadSizeMask]
		nr, er := r.Read(payloadBuf)

		if nr > 0 {
			n += int64(nr)
			buf = buf[:2+w.Overhead()+nr+w.Overhead()]
			payloadBuf = payloadBuf[:nr]
			buf[0], buf[1] = byte(nr>>8), byte(nr) // big-endian payload size
			w.Seal(buf[:0], w.nonce, buf[:2], nil) // 负责加密的
			increment(w.nonce)

			w.Seal(payloadBuf[:0], w.nonce, payloadBuf, nil) // 负责加密的
			increment(w.nonce)

			_, ew := w.Writer.Write(buf) // 写入
			if ew != nil {
				err = ew
				break
			}
		}

		if er != nil {
			if er != io.EOF { // ignore EOF as per io.ReaderFrom contract
				err = er
			}
			break
		}
	}

	return n, err
}
//...
```

其实还挺好懂.

接下来看 这个 Seal :

```go
	// Seal encrypts and authenticates plaintext, authenticates the
	// additional data and appends the result to dst, returning the updated
	// slice. The nonce must be NonceSize() bytes long and unique for all
	// time, for a given key.
	//
	// To reuse plaintext's storage for the encrypted output, use plaintext[:0]
	// as dst. Otherwise, the remaining capacity of dst must not overlap plaintext.
	Seal(dst, nonce, plaintext, additionalData []byte) []byte
```

注释赛高! 不过这里已经不是这个包里面的内容了. 是一些官方提供的加密解密库了.

那么想过去 Read 的流程也不会有什么太大变化了.

就不看了吧.

那么我们现在来处理一下遗留问题.

1. [这里](https://github.com/shadowsocks/go-shadowsocks2/blob/master/shadowaead/stream.go#L252) 的这行代码是什么意思呢?

简单看了一下, 发现是记录一下之前用过的盐, 防止盐重复的.

不是什么很难懂的东西.
