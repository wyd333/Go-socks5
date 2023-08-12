# Go-socks5
Go语言实现简单的socks5代理服务器



# SOCKS5代理服务器

对于大家来说，一提到代理服务器，第一想到的是翻墙。不过遗憾的是， socks5 协议它虽然是代理协议，但它并不能用来翻墙，它的协议都是明文传输。这个协议历史比较久远，诞生于互联网早期。

它的用途是，比如某些企业的内网为了确保安全性，有很严格的防火墙策略，但是带来的副作用是访问某些资源会很麻烦。socks5 相当于在防火墙开了个口子，让授权的用户可以通过单个端口去访问内部的所有资源。实际上很多翻墙软件，最终暴露的也是一个 socks5 协议的端口。

在爬虫开发中，爬虫的爬取过程中很容易会遇到IP访问频率超过限制的问题，这个时候很多人就会去网上找一些代理IP池，这些代理IP池里面的很多代理的协议也就是 socks5。

## **socks5协议的原理

![img](https://pqmo9n8d6gt.feishu.cn/space/api/box/stream/download/asynccode/?code=YzVjNmM3ZGRkY2YwNGZiOGU5N2MxZWMzZjM1N2MyNTZfZ05aVWJJQ09KUVc2T0pYRHN1ak1oeEIxZFp4V3JqQVVfVG9rZW46U0xuTWJ6Z0xTbzRKR3d4UERSNGM3eDJOblRiXzE2OTE4Mzk5NDk6MTY5MTg0MzU0OV9WNA)

正常浏览器访问一个网站，如果不经过代理服务器的话，会先和对方的网站建立 TCP 连接，然后三次握手。握手完之后发起 HTTP 请求，然后服务返回 HTTP 响应。

如果设置代理服务器之后，流程会变得复杂一些：

- 首先是浏览器和 socks5 代理建立 TCP 连接，代理再和真正的服务器建立 TCP 连接。这里可以分成四个阶段：
  - 握手阶段
  - 认证阶段
  - 请求阶段
  - relay 阶段
- 第一个握手阶段，浏览器会向 socks5 代理发送请求，包的内容包括一个协议的版本号和支持的认证的种类。
- socks5 服务器会选中一个认证方式，返回给浏览器。如果返回的是 00 的话就代表不需要认证，返回其他类型的话会开始认证流程。
- 第三个阶段是请求阶段，认证通过之后浏览器会 socks5 服务器发起请求。主要信息包括版本号，请求的类型。一般主要是 connection 请求，代表代理服务器要和某个域名或者某个 IP 地址某个端口建立 TCP 连接。代理服务器收到响应之后，会真正和后端服务器建立连接，然后返回一个响应。
- 第四个阶段是 relay 阶段。此时浏览器会发送 正常发送请求，然后代理服务器接收到请求之后，会直接把请求转换到真正的服务器上。然后如果真正的服务器以后返回响应的话，那么也会把请求转发到浏览器这边。实际代理服务器并不关心流量的细节，可以是 HTTP流量，也可以是其它 TCP 流量。

这个就是 socks5 协议的工作原理，接下来我们尝试去简单地实现它。

## 1、TCP echo server

在实现代理服务器之前，我们先实现一个简单的回显服务器，以测试我们的server写的对不对。

当运行时，此代码将创建一个简单的 TCP 服务器，监听在 127.0.0.1 的 1080 端口上，接受客户端连接并将客户端发送的数据原样返回。以下是逐句加注释的代码解释：



```Go
package main

import (
        "bufio"
        "log"
        "net"
)

func main() {
        // 在本地的 127.0.0.1:1080 地址上创建一个 TCP 服务器
        server, err := net.Listen("tcp", "127.0.0.1:1080")
        if err != nil {
                panic(err)
        }
        // 循环等待客户端连接
        for {
                // 接受客户端的连接
                client, err := server.Accept()
                if err != nil {
                        log.Printf("Accept failed %v", err)
                        continue
                }
                // 启动一个新的 goroutine 处理客户端连接
                go process(client)
        }
}

func process(conn net.Conn) {
        // 在函数退出时关闭客户端连接
        defer conn.Close()
        // 创建一个用于读取客户端数据的 bufio.Reader
        reader := bufio.NewReader(conn)
        for {
                // 从客户端读取一个字节
                b, err := reader.ReadByte()
                if err != nil {
                        break
                }
                // 将读取的字节原样发送回客户端
                _, err = conn.Write([]byte{b})
                if err != nil {
                        break
                }
        }
}
```

这是一个基本的 TCP 服务器，用于接受客户端连接并将接收到的数据返回给客户端。每当客户端发送一个字节，服务器会将相同的字节返回。

我们通过`nc`命令来进行测试。首先需要下载netcat，地址及教程：https://blog.csdn.net/BoomLee/article/details/102563472

安装配置完毕后，Goland中启动echo-server程序（直接启动或用命令`go run echo-server.go`）。启动后，另开一个cmd窗口，在其中输入nc命令：`nc 127.0.0.1 1080`

输入什么就显示什么，一个简单的回显服务器就搞定了。

![img](https://pqmo9n8d6gt.feishu.cn/space/api/box/stream/download/asynccode/?code=NjFhY2Q5OThiNjBkMjZkZGU5M2U1YWRmYzlhM2ZhZGVfQWw0bVdKY3ZLOHhmSXl0Ukp0bTZGOTVTSmh3QVE0aUVfVG9rZW46Q1FocGJYODRZb0NWY0d4cnJSc2N6WFhCbnNlXzE2OTE4Mzk5NDk6MTY5MTg0MzU0OV9WNA)

## 2、认证阶段：auth

```Go
package main

import (
        "bufio"
        "fmt"
        "io"
        "log"
        "net"
)

const socks5Ver = 0x05
const cmdBind = 0x01
const atypeIPV4 = 0x01
const atypeHOST = 0x03
const atypeIPV6 = 0x04

func main() {
        server, err := net.Listen("tcp", "127.0.0.1:1080") // 在 127.0.0.1 的 1080 端口上创建 TCP 服务器
        if err != nil {
                panic(err)
        }
        for {
                client, err := server.Accept() // 接受客户端连接
                if err != nil {
                        log.Printf("Accept failed %v", err)
                        continue
                }
                go process(client) // 启动一个新的 goroutine 处理客户端连接
        }
}

func process(conn net.Conn) {
        defer conn.Close() // 在函数结束时关闭客户端连接
        reader := bufio.NewReader(conn)
        err := auth(reader, conn) // 进行客户端认证
        if err != nil {
                log.Printf("client %v auth failed:%v", conn.RemoteAddr(), err)
                return
        }
        log.Println("auth success")
}




// +----+----------+----------+
// |VER | NMETHODS | METHODS  |
// +----+----------+----------+
// | 1  |    1     | 1 to 255 |
// +----+----------+----------+
// VER: 协议版本，socks5为0x05
// NMETHODS: 支持认证的方法数量
// METHODS: 对应NMETHODS，NMETHODS的值为多少，METHODS就有多少个字节。RFC预定义了一些值的含义，内容如下:
// X’00’ NO AUTHENTICATION REQUIRED
// X’02’ USERNAME/PASSWORD
func auth(reader *bufio.Reader, conn net.Conn) (err error) {
        // 解析客户端发送的认证请求
        ver, err := reader.ReadByte() // 读取协议版本
        if err != nil {
                return fmt.Errorf("read ver failed:%w", err)
        }
        if ver != socks5Ver {
                return fmt.Errorf("not supported ver:%v", ver)
        }
        methodSize, err := reader.ReadByte() // 读取支持的认证方法数量
        if err != nil {
                return fmt.Errorf("read methodSize failed:%w", err)
        }
        method := make([]byte, methodSize)
        _, err = io.ReadFull(reader, method) // 读取支持的认证方法列表
        if err != nil {
                return fmt.Errorf("read method failed:%w", err)
        }
        log.Println("ver", ver, "method", method)

        // 发送认证响应给客户端
        // +----+--------+
        // |VER | METHOD |
        // +----+--------+
        // | 1  |   1    |
        // +----+--------+
        _, err = conn.Write([]byte{socks5Ver, 0x00}) // 发送无需认证的响应给客户端
        if err != nil {
                return fmt.Errorf("write failed:%w", err)
        }
        return nil
}
```

这段代码演示了一个简单的 SOCKS5 服务器，用于处理客户端的认证请求，并向客户端发送响应。现在我们用 curl 命令进行一下测试。首先还是一样，在Goland运行项目程序。然后在另一个终端执行curl命令：

```Go
curl --socks5 127.0.0.1:1080 -v http://www.qq.com
```

此时curl 命令肯定是不成功的，因为协议还没实现完成。但是看日志会发现， version和method 可以正常打印，说明当前我们的实现是正确的。

![img](https://pqmo9n8d6gt.feishu.cn/space/api/box/stream/download/asynccode/?code=MjQwNTNjNTgxZGFmOTMzOTYxM2FiNGVhM2RhMDc3ZmFfVGNOMDBua0RCck1PeHYzUHlYR3cxakZtZFpXQmx2VEhfVG9rZW46WU1nUWJIanUyb2FiRXV4MXZMMWM2UWdibldoXzE2OTE4Mzk5NDk6MTY5MTg0MzU0OV9WNA)

![img](https://pqmo9n8d6gt.feishu.cn/space/api/box/stream/download/asynccode/?code=NTY1YWY0NzRiZWQ2ZDFhYzUwNWFkYjAwMThlMjBmODdfaWUyWUxkb2dJV0NZaG1pcHRXaFN6MUtabTJZWEFRQXFfVG9rZW46Q01wVWJQcUZQb2FTSFp4NkFmcGNMVnhhbmlmXzE2OTE4Mzk5NDk6MTY5MTg0MzU0OV9WNA)

## 3、请求阶段

接下来我们开始做第三步：请求阶段。

我们试图读取到携带 URL 或 IP 地址+端口的包，然后把它打印出来。auth 函数和 connect 函数类似，同样在 process 里去调用。

再实现 connect 函数的代码。根据请求阶段的逻辑，浏览器会发送一个包，包里面包含如下6个字段：

1. VER 版本号，socks5的值为0x05
2. CMD 0x01表示CONNECT请求
3. RSV 保留字段，值为0x00
4. ATYP 目标地址类型，DST.ADDR的数据对应这个字段的类型。
   1. 0x01表示IPv4地址，DST.ADDR为4个字节
   2. 0x03表示域名，DST.ADDR是一个可变长度的域名
5. DST.ADDR 一个可变长度的值
6. DST.PORT 目标端口，固定2个字节

接下来我们要挨个去把这6个字段读出来。

面这四个字段总共四个字节，我们可以一次性把它读出来。创建一个长度为4的缓冲区，然后用io.ReadFull()把它整个填充满。

```Go
//connect()

buf := make([]byte, 4)
_, err = io.ReadFull(reader, buf)
if err != nil {
    return fmt.Errorf("read header failed:%w", err)
}
```

这样就能一次性读取到前面4个字段，它们是定长的。对于每个字段，都要验证合法性：

```Go
//connect()

ver, cmd, atyp := buf[0], buf[1], buf[3]
if ver != socks5Ver {
    return fmt.Errorf("not supported ver:%v", ver)
}
if cmd != cmdBind {
    return fmt.Errorf("not supported cmd:%v", cmd)
}
addr := ""
switch atyp {
case atypeIPV4:
    _, err = io.ReadFull(reader, buf)
    if err != nil {
       return fmt.Errorf("read atyp failed:%w", err)
    }
    addr = fmt.Sprintf("%d.%d.%d.%d", buf[0], buf[1], buf[2], buf[3])
case atypeHOST:
    hostSize, err := reader.ReadByte()
    if err != nil {
       return fmt.Errorf("read hostSize failed:%w", err)
    }
    host := make([]byte, hostSize)
    _, err = io.ReadFull(reader, host)
    if err != nil {
       return fmt.Errorf("read host failed:%w", err)
    }
    addr = string(host)
case atypeIPV6:    //这个暂时不实现
    return errors.New("IPv6: no supported yet")
default:
    return errors.New("invalid atyp")
}
```

最后还有两个字节是 port ，我们读取它，然后按协议规定的大端字节序转换成数字。

由于上面的 buffer 已经不会被其他变量使用了，我们可以直接复用之前的内存，建立一个临时的 slice ，长度是2，用于读取，这样的话最多会只读两个字节回来。 接下来我们把这个地址和端口打印出来用于调试。

```Go
//connect()

_, err = io.ReadFull(reader, buf[:2])
if err != nil {
    return fmt.Errorf("read port failed:%w", err)
}
port := binary.BigEndian.Uint16(buf[:2])

log.Println("dial", addr, port)
```

收到浏览器的这个请求包之后，我们需要返回一个包，这个包有很多字段，但其实大部分都不会使用：

![img](https://pqmo9n8d6gt.feishu.cn/space/api/box/stream/download/asynccode/?code=MWEwZWRhOWYzMTM0ZTkyNmE5ZWRkZmRhYWVmNzMxZDRfZGtCUGtEZGRTWk04bG9qT01Ma0J6SjF1dTBHS2g2SHRfVG9rZW46RjFIR2JkYVBOb2gwRDJ4RUo4ZWN4cXRKbkhnXzE2OTE4Mzk5NDk6MTY5MTg0MzU0OV9WNA)

运行测试：虽然还没有完全成功，但是能够打印出IP地址和端口号了，说明实验还是成功的。

![img](https://pqmo9n8d6gt.feishu.cn/space/api/box/stream/download/asynccode/?code=MzRmYjBlNjg0NGYxYzViNDk3M2JhMjk3YTBkZDViZTZfeTY3aG9HdVY3enZsRVZZUFhqWTFSajNSSm5Kd0gwSEVfVG9rZW46VmpodGJWZWh1b09zRkJ4UXNERWN0VHdrbkxkXzE2OTE4Mzk5NDk6MTY5MTg0MzU0OV9WNA)

## 4、relay阶段

直接用 net.Dial 建立一个 TCP 连接。建立完连接之后，同样要加一个 defer 来关闭连接。

接下来需要建立浏览器和下游服务器的双向数据转发。标准库的 io.Copy 可以实现一个单向数据转发。完成双向转发还需启动两个 goroutinue。

### 完整代码如下

```Go
package main

import (
    "bufio"
    "context"
    "encoding/binary"
    "errors"
    "fmt"
    "io"
    "log"
    "net"
)

const socks5Ver = 0x05
const cmdBind = 0x01
const atypeIPV4 = 0x01
const atypeHOST = 0x03
const atypeIPV6 = 0x04

func main() {
    server, err := net.Listen("tcp", "127.0.0.1:1080")
    if err != nil {
       panic(err)
    }
    for {
       client, err := server.Accept()
       if err != nil {
          log.Printf("Accept failed %v", err)
          continue
       }
       go process(client)
    }
}

func process(conn net.Conn) {
    defer conn.Close()
    reader := bufio.NewReader(conn)
    err := auth(reader, conn)
    if err != nil {
       log.Printf("client %v auth failed:%v", conn.RemoteAddr(), err)
       return
    }
    err = connect(reader, conn)
    if err != nil {
       log.Printf("client %v auth failed:%v", conn.RemoteAddr(), err)
       return
    }
}

func auth(reader *bufio.Reader, conn net.Conn) (err error) {
    // +----+----------+----------+
    // |VER | NMETHODS | METHODS  |
    // +----+----------+----------+
    // | 1  |    1     | 1 to 255 |
    // +----+----------+----------+
    // VER: 协议版本，socks5为0x05
    // NMETHODS: 支持认证的方法数量
    // METHODS: 对应NMETHODS，NMETHODS的值为多少，METHODS就有多少个字节。RFC预定义了一些值的含义，内容如下:
    // X’00’ NO AUTHENTICATION REQUIRED
    // X’02’ USERNAME/PASSWORD

    ver, err := reader.ReadByte()
    if err != nil {
       return fmt.Errorf("read ver failed:%w", err)
    }
    if ver != socks5Ver {
       return fmt.Errorf("not supported ver:%v", ver)
    }
    methodSize, err := reader.ReadByte()
    if err != nil {
       return fmt.Errorf("read methodSize failed:%w", err)
    }
    method := make([]byte, methodSize)
    _, err = io.ReadFull(reader, method)
    if err != nil {
       return fmt.Errorf("read method failed:%w", err)
    }

    // +----+--------+
    // |VER | METHOD |
    // +----+--------+
    // | 1  |   1    |
    // +----+--------+
    _, err = conn.Write([]byte{socks5Ver, 0x00})
    if err != nil {
       return fmt.Errorf("write failed:%w", err)
    }
    return nil
}

func connect(reader *bufio.Reader, conn net.Conn) (err error) {
    // +----+-----+-------+------+----------+----------+
    // |VER | CMD |  RSV  | ATYP | DST.ADDR | DST.PORT |
    // +----+-----+-------+------+----------+----------+
    // | 1  |  1  | X'00' |  1   | Variable |    2     |
    // +----+-----+-------+------+----------+----------+
    // VER 版本号，socks5的值为0x05
    // CMD 0x01表示CONNECT请求
    // RSV 保留字段，值为0x00
    // ATYP 目标地址类型，DST.ADDR的数据对应这个字段的类型。
    //   0x01表示IPv4地址，DST.ADDR为4个字节
    //   0x03表示域名，DST.ADDR是一个可变长度的域名
    // DST.ADDR 一个可变长度的值
    // DST.PORT 目标端口，固定2个字节

    buf := make([]byte, 4)
    _, err = io.ReadFull(reader, buf)
    if err != nil {
       return fmt.Errorf("read header failed:%w", err)
    }
    ver, cmd, atyp := buf[0], buf[1], buf[3]
    if ver != socks5Ver {
       return fmt.Errorf("not supported ver:%v", ver)
    }
    if cmd != cmdBind {
       return fmt.Errorf("not supported cmd:%v", cmd)
    }
    addr := ""
    switch atyp {
    case atypeIPV4:
       _, err = io.ReadFull(reader, buf)
       if err != nil {
          return fmt.Errorf("read atyp failed:%w", err)
       }
       addr = fmt.Sprintf("%d.%d.%d.%d", buf[0], buf[1], buf[2], buf[3])
    case atypeHOST:
       hostSize, err := reader.ReadByte()
       if err != nil {
          return fmt.Errorf("read hostSize failed:%w", err)
       }
       host := make([]byte, hostSize)
       _, err = io.ReadFull(reader, host)
       if err != nil {
          return fmt.Errorf("read host failed:%w", err)
       }
       addr = string(host)
    case atypeIPV6:
       return errors.New("IPv6: no supported yet")
    default:
       return errors.New("invalid atyp")
    }
    _, err = io.ReadFull(reader, buf[:2])
    if err != nil {
       return fmt.Errorf("read port failed:%w", err)
    }
    port := binary.BigEndian.Uint16(buf[:2])

    dest, err := net.Dial("tcp", fmt.Sprintf("%v:%v", addr, port))
    if err != nil {
       return fmt.Errorf("dial dst failed:%w", err)
    }
    defer dest.Close()
    log.Println("dial", addr, port)

    // +----+-----+-------+------+----------+----------+
    // |VER | REP |  RSV  | ATYP | BND.ADDR | BND.PORT |
    // +----+-----+-------+------+----------+----------+
    // | 1  |  1  | X'00' |  1   | Variable |    2     |
    // +----+-----+-------+------+----------+----------+
    // VER socks版本，这里为0x05
    // REP Relay field,内容取值如下 X’00’ succeeded
    // RSV 保留字段
    // ATYPE 地址类型
    // BND.ADDR 服务绑定的地址
    // BND.PORT 服务绑定的端口DST.PORT
    _, err = conn.Write([]byte{0x05, 0x00, 0x00, 0x01, 0, 0, 0, 0, 0, 0})
    if err != nil {
       return fmt.Errorf("write failed: %w", err)
    }
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go func() {
       _, _ = io.Copy(dest, reader)
       cancel()
    }()
    go func() {
       _, _ = io.Copy(conn, dest)
       cancel()
    }()

    <-ctx.Done()
    return nil
}
```

![img](https://pqmo9n8d6gt.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmFlMTc1YmE5Yjg5NDJmZjlhY2QxYjNlYTJjMjU1ZDJfNktlQnhsRDJuU3pLaXRRYTEyOXFXUFF5VHNVQUZLN2NfVG9rZW46TjZqVGJXNkFzb3k2U3F4MHVVOWMyREc5bnJoXzE2OTE4Mzk5NDk6MTY5MTg0MzU0OV9WNA)

到此，socks5代理服务器就实现完成了。

也可以在浏览器中进行测试。Chrome浏览器只需安装SwitchyOmega插件：

![img](https://pqmo9n8d6gt.feishu.cn/space/api/box/stream/download/asynccode/?code=MzAyNWJkYTE2ZDY2ODZkOTkwZWJmYjVjNWU2MTc5OTlfd1FLdGhDb1JoSU1OM1dyVW1NN0EybnZucUllTTV3RzdfVG9rZW46VDBkWGJaa2Myb1NOakF4NXZZa2NUVmY1bktkXzE2OTE4Mzk5NDk6MTY5MTg0MzU0OV9WNA)

可以在浏览器里面再测试一下，在插件中新建一个情景模式， 代理服务器选 socks5，端口 1080 ，保存并启用。此时你正常地访问其它网站，代理服务器这边会显示出浏览器版本的域名和端口。

![img](https://pqmo9n8d6gt.feishu.cn/space/api/box/stream/download/asynccode/?code=M2FlZTg5NmMyZDNmNjM3ZjljNTlhNzczYTRiNGE4NzdfQWRxRUNET1d4SXlqM1F2SzRPYjIzaHhZdkxLMHFqUlhfVG9rZW46RmE4TGJCZjVTb1BiMEh4VDVyQWNDRDV4bnVlXzE2OTE4Mzk5NDk6MTY5MTg0MzU0OV9WNA)