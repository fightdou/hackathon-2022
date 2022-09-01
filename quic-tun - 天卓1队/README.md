# quic-tun

### 介绍

一个快速且安全的 TCP 隧道工具，能加速弱网环境下（如网络有丢包）TCP 的转发性能。

#### 原理概述

quic-tun 项目包含两个程序，quictun-server 和 quictun-client，其核心原理可以用一句话概述：quictun-server 把 server 端 TCP/UNIX-SOCKET 应用的传输层协议转为 [QUIC](https://www.chromium.org/quic/)，
然后 quictun-client 再把 QUIC 转为 TCP/UNIX-SOCKET，客户端应用程序只需要连接到 quictun-client 服务就可以和 server 端应用程序交互了。

因为 quictun-server 和 quictun-client 之间的网络传输使用的是 QUIC 协议，所以拥有了 QUIC 协议的所有能力，包括重传算法和拥塞控制算法等，进而能够轻松的应对复杂网络环境。因此对于整体的网络状况会有极大优化。关于 quic-tun 对网络传输的优化，我们做了一些测试，并编写了[测试报告](https://kungze.github.io/documents/quic-tun/performance-test/)，
想了解更多信息，请[点击查看](https://kungze.github.io/documents/quic-tun/performance-test/)。

#### 特点摘要

- 客户端服务端采用 QUIC 协议通信
- 客户端服务端之间的数据加密传输
- 优化网络传输
- 公网环境下仅映射服务端地址，并通过客户端建立隧道即可访问服务端内部所有服务
- 支持为 TCP/UNIX-SOCKET 应用程序建立隧道，支持 TCP/UNIX-SOCKET 应用相互转换
- 能够识别 spice 协议
- 可运行在 Linux、windows 64-bit 系统上
- 多线程、使用所有的 CPU
- 没有外部依赖，纯粹的 GO 语言编写
- 支持命令行和配置文件

为了能够让更多的开源爱好者参与进来，我们把 [quic-tun](https://github.com/kungze/quic-tun) 开源。

同时我们希望通过这次参赛的机会，能够得到各位专家评委老师的指点和建议，以及大家的关注。

#### 软件架构

quic-tun 不仅有优化 TCP 传输的作用，他还能把 TCP 应用转为 UNIX-SOCKET 应用，UNIX-SOCKET 应用转为 TCP 应用，其架构图如下：

![quic-tun](https://lplearn-note-pic.oss-cn-beijing.aliyuncs.com/note/2022/quic-tun.png)

#### 概念解释

* **tunnel**：隧道，`quic-tun`会为每个 TCP 连接创建一个 tunnel，一个 tunnel 对应一个 QUIC 的 `connection`（quic-tun 为实现多路复用提出的一个概念）。
* **client endpoint**：隧道的 client 端点，监听在 TCP 端口或者 UNIX-SOCKET 文件，用于接受 client 应用程序的请求并与 server endpoint 建立隧道。
* **server endpoint**：隧道的 server 端点，监听在 UDP(quic) 端口，与 client endpoint 建立隧道后把隧道传输的数据转发到 server 应用。
* **token**：用于告诉 server endpoint，client 应用程序需要连接到哪个 server 应用程序，在 client endpoint 接受 client 应用程序的连接
  后第一件事就是生成 token，然后把这个 token 发送到 server endpoint，server endpoint 从这个 token 中解析出 server 应用程序的地址，然后连接到应用
  程序，在然后与 client endpoint 建立隧道。quic-tun 提供了很多 token 的获取和解析插件，想了解关于 token 更多信息，请阅读我们[专门的章节](https://kungze.github.io/documents/quic-tun/token/)

### 使用说明

调整内核参数，增大缓存区，详细原因，请参考[官方文档](https://github.com/lucas-clemente/quic-go/wiki/UDP-Receive-Buffer-Size)。
server 端和 client 端都要调整这个参数。

```console
sysctl -w net.core.rmem_max=2500000
```

打开 [release 页面](https://github.com/kungze/quic-tun/releases)，下载最新版本的 quic-tun，并解压

二进制包可直接从下面的连接获取(该包我们已经放在 bin 目录下, 可直接取用)

```console
# linux
wget https://github.com/kungze/quic-tun/releases/download/v0.0.4/quic-tun_0.0.4_linux_amd64.tar.gz

# windows
wget https://github.com/kungze/quic-tun/releases/download/v0.0.4/quic-tun_0.0.4_windows_amd64.tar.gz
```

```console
tar xvfz quic-tun_0.0.4_linux_amd64.tar.gz
```

启动 server 端程序

```console
./quictun-server --listen-on 172.18.31.36:7500
```

启动客户端程序

```console
./quictun-client --listen-on tcp:127.0.0.1:6500 --server-endpoint 172.18.31.36:7500 --token-source tcp:172.18.30.117:22
```

**注意**：上面参数 `--token-source` 指定一个 token，这个 token 用于告诉 server 端客户端应用程序想要连接到哪个应用程序。想了解关于 token 更多信息，请阅读[token](https://kungze.github.io/documents/quic-tun/token/)

这个时候我们可是使用 `ssh` 命令测试

```console
$ ssh root@127.0.0.1 -p 6500
root@127.0.0.1's password:
```

可以使用 `--help` 查看更多 `quictun-server` 和 `quictun-client` 支持的更多命令行参数。

### 测试数据

为了验证 quic-tun 对网络传输性能的影响，我们做了一些测试，通过下面的折线图能够看出 quic-tun 对于网络性能的提升还是很大的。
想了解更多信息以及测试方法，请[点击查看](https://kungze.github.io/documents/quic-tun/performance-test/)。

![q1](https://lplearn-note-pic.oss-cn-beijing.aliyuncs.com/note/2022/20220901155620.png)

![q2](https://lplearn-note-pic.oss-cn-beijing.aliyuncs.com/note/2022/20220901155635.png)

![q3](https://lplearn-note-pic.oss-cn-beijing.aliyuncs.com/note/2022/20220901220651.png)

### 特性介绍

#### Token 插件
[Token plugin](https://kungze.github.io/documents/quic-tun/token/)
#### restful API

quic-tun 还提供了 restful API。通过这些API，你可以查询正在运行的隧道的信息。你可以在启动 quictun-server/quictun-client 时通过 
--httpd-listen-on 设置 API 服务的监听地址，如下所示。

```console
./quictun-server --httpd-listen-on 127.0.0.1:18086
```

然后，你可以使用 curl 命令来查询所有活动的隧道，如下所示。

```console
$ curl http://127.0.0.1:18086/tunnels | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   227  100   227    0     0   221k      0 --:--:-- --:--:-- --:--:--  221k
[
  {
    "uuid": "2e1ce596-8357-4a46-aef1-0c4871b893cd",
    "streamId": 4,
    "endpoint": "server",
    "serverAppAddr": "172.18.31.97:22",
    "remoteEndpointAddr": "172.18.29.161:46706",
    "createdAt": "2022-06-21 11:44:05.074778434 +0800 CST m=+86.092908233",
    "protocol": "",
    "protocolProperties": null
  }
]
```

此外，我们实现了一个 [spice](https://www.spice-space.org/spice-protocol.html) 协议鉴别器，它可以从通过隧道的流量中提取更多关于 
spice 的属性。因此，对于 spice 应用程序，调用查询 API，你可以得到以下响应。

```console
# curl http://172.18.29.161:18086/tunnels | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  5137    0  5137    0     0  2508k      0 --:--:-- --:--:-- --:--:-- 5016k
[
  {
    "uuid": "9eb73491-ef38-463d-85c3-d4512152d224",
    "streamId": 0,
    "endpoint": "server",
    "serverAppAddr": "172.18.11.2:5915",
    "remoteEndpointAddr": "172.18.29.161:56465",
    "createdAt": "2022-06-21 11:41:28.85774404 +0800 CST m=+47.535828999",
    "protocol": "spice",
    "protocolProperties": {
      "version": "2.2",
      "sessionId": "d0306d75",
      "channelType": "main",
      "serverName": "instance-e548a827-8937-4047-a756-e56937017128",
      "serverUUID": "e548a827-8937-4047-a756-e56937017128"
    }
  },
  {
    "uuid": "66bad84d-318c-4e14-b3be-a5cb796e7f61",
    "streamId": 44,
    "endpoint": "server",
    "serverAppAddr": "172.18.11.2:5915",
    "remoteEndpointAddr": "172.18.29.161:56465",
    "createdAt": "2022-06-21 11:41:28.937090895 +0800 CST m=+47.615175866",
    "protocol": "spice",
    "protocolProperties": {
      "version": "2.2",
      "sessionId": "d0306d75",
      "channelType": "record"
    }
  },
  {
    "uuid": "ff93728e-38fb-435c-8728-3ece51077b95",
    "streamId": 56,
    "endpoint": "server",
    "serverAppAddr": "172.18.11.2:5915",
    "remoteEndpointAddr": "172.18.29.161:56465",
    "createdAt": "2022-06-21 11:41:29.224234488 +0800 CST m=+47.902319441",
    "protocol": "spice",
    "protocolProperties": {
      "version": "2.2",
      "sessionId": "d0306d75",
      "channelType": "inputs"
    }
  },
  {
    "uuid": "fbfd963c-e6b9-4c13-bfec-8965b1c56851",
    "streamId": 12,
    "endpoint": "server",
    "serverAppAddr": "172.18.11.2:5915",
    "remoteEndpointAddr": "172.18.29.161:56465",
    "createdAt": "2022-06-21 11:41:28.93269002 +0800 CST m=+47.610774997",
    "protocol": "spice",
    "protocolProperties": {
      "version": "2.2",
      "sessionId": "d0306d75",
      "channelType": "usbredir"
    }
  },
  {
    "uuid": "62fa355c-9c0d-4dbb-9e84-2b0c354cf8cc",
    "streamId": 48,
    "endpoint": "server",
    "serverAppAddr": "172.18.11.2:5915",
    "remoteEndpointAddr": "172.18.29.161:56465",
    "createdAt": "2022-06-21 11:41:28.937563866 +0800 CST m=+47.615648836",
    "protocol": "spice",
    "protocolProperties": {
      "version": "2.2",
      "sessionId": "d0306d75",
      "channelType": "display"
    }
  },
  {
    "uuid": "ce2e0bef-0ccb-4325-ab65-a6d3783c47ae",
    "streamId": 52,
    "endpoint": "server",
    "serverAppAddr": "172.18.11.2:5915",
    "remoteEndpointAddr": "172.18.29.161:56465",
    "createdAt": "2022-06-21 11:41:29.223947759 +0800 CST m=+47.902032695",
    "protocol": "spice",
    "protocolProperties": {
      "version": "2.2",
      "sessionId": "d0306d75",
      "channelType": "cursor"
    }
  },
  {
    "uuid": "c5169c4a-ab69-406b-b36a-68c0ab7d9d7f",
    "streamId": 40,
    "endpoint": "server",
    "serverAppAddr": "172.18.11.2:5915",
    "remoteEndpointAddr": "172.18.29.161:56465",
    "createdAt": "2022-06-21 11:41:28.936673702 +0800 CST m=+47.614758657",
    "protocol": "spice",
    "protocolProperties": {
      "version": "2.2",
      "sessionId": "d0306d75",
      "channelType": "playback"
    }
  }
]
```

### 使用到的 google 技术

[quic协议](https://baike.baidu.com/item/QUIC/17341272?fr=aladdin)

[go语言](https://baike.baidu.com/item/go/953521?fromtitle=Golang&fromid=2215139&fr=aladdin)

### 团队介绍

杨建峰
yangjianfeng@troila.com

刘鹏
liupeng2@troila.com

刘慧敏
liuhuimin@troila.com

万学军
wanxuejun@troila.com
