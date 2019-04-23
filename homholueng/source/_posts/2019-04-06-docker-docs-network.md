---
title: Docker 网络基础
tags:
  - Docker
  - Network
categories:
  - Docker
date: 2019-04-06 18:17:19
---


Docker 容器和服务的功能之所以如此强大，得益于其能够通过网络与其他组件进行互联。与此同时，容器和服务也不需要关心自身和与其互联的组件是否是部署在 Docker 上的，也不需要关心 Docker 宿主机的操作系统类型，因为 Docker 已经帮我们处理了这些事情，我们能够使用一种平台无关的方式来管理这些组件和配置。

## 网络驱动

Docker 的网络系统通过驱动来实现了可插拔的特性，并且默认提供了以下几种类型的驱动：

- `bridge`：默认使用的驱动类型，**一般独立的容器需要进行网络通信时都会使用这种驱动**。
- `host`：对于独立容器来说，host 驱动去除了 Docker 宿主机与容器之间的网络隔离，直接使用宿主机网络。该模式仅仅对 17.06 之后的版本中的 swarm services 有效。
- `overlay`：overlay 驱动将多个 Docker 连接起来使得 swarm services 之间能够进行互相通信。同时，你也可以使用 overlay 网络来让 swarm services 与独立容器、或是运行在不同的 Docker 进程上的独立容器与独立容器之间进行网络通信。
- `macvlan`：macvlan 驱动能够为容器设置 MAC 地址，使其在网络中表现的与真实的物理设备一样。当你要处理一些需要直连物理网络的遗留应用时，macvlan 或许是比较好的选择。
- `none`：使用的这种插件的容器将无法进行网络通信，swarm services 无法使用这种类型的插件。

除了上面列出的这些插件之外，我们还可以按需选择从 [Docker Hub](https://hub.docker.com/search/?category=network&q=&type=plugin) 处获取的第三方网络插件。

### 总结

- 当你需要让同一个 Docker 宿主机下的多个容器进行网络通信时，**用户定义的 bridge 模式**是最好的选择。
- 当你需要让运行在多个不同的 Docker 宿主机上的容器、或是多个 swarm services 进行网络通信时，**overlay 模式**是最好的选择。
- 当你希望你的容器看起来像是一个真实的物理设备时，**macvlan 模式**是最好的选择。
- 当你需要将一些特殊的网络栈集成到 Docker 中时，**第三方网络插件**或许能满足你的需求。

## 使用 bridge 网络

在计算机网络的术语中，桥接网络（bridge network，后文称为 bridge 网络）指的是在网络段之间转发流量的链路层设备。网桥（bridge）既可以是硬件设备，也可以是运行在主机内核中的软件。

在 Docker 的术语中，bridge 网络通过软件层的网桥使得连接到该网络上的容器能够进行通信，并对连接到不同网络中的容器进行隔离。Docker 的 bridge 驱动会自动将规则配置到宿主机中使得不同 bridge 网络中的容器无法直接与对方进行通信。

但是，bridge 网络只能应用在运行在同一台宿主机上的容器上。当 Docker 启动时，其会自动创建一个默认的 bridge 网络，并且新启动的容器在没有指定特定网络的前提下会自动加入该默认的 bridge 网络中。**但用户定义的 bridge 网络比默认创建的 bridge 网络更加强大。**

### 用户定义的与默认的 bridge 网络的差异

- 用户定义的 bridge 网络相较与默认的能在容器化的应用间提供更好的隔离性和互操作性
    - 用户定义的 bridge 网络中的容器会自动互相暴露所有的端口，并向外部屏蔽所有的端口
- 用户定义的 bridge 网络能够自动的在容器间进行 DNS 解析
    - 用户定义的 bridge 网络间的容器能够通过网络中其他容器的名称来进行通信
- 容器能在运行中随时加入用户定义的 bridge 网络或从其中脱离出来
    - 若要将容器从默认 bridge 网络中脱离出来，需要先将容器停止
- 每一个用户定义的 bridge 网络都创建了一个可配置的网桥
    - 若要配置默认 bridge 网络，需要在 Docker 外进行配置，并且需要重启 Docker；而用户定义的 bridge 网络通过 `docker network create` 进行创建和配置，如果不同的应用对网络有不同的要求，只需要在创建时单独配置每个网络即可。
- 连接到默认 bridge 网络中的容器会共享环境变量
    - 在用户定义的 bridge 网络中需要通过其他方式来共享环境变量，例如：挂载公共目录，使用 `docker-compose` 来管理容器和共享变量

> 当我们创建用户定义网络、将容器连接到网络或是从中断开时，容器会操作操作系统底层的网络设施，例如添加网桥设备或是配置 Linux 中的 `iptables`。

bridge 网络相关操作请参考官方文档：[Use bridge networks](https://docs.docker.com/network/bridge/)

### bridge 网络通信实践

#### 使用默认的 bridge 网络

> 这种类型的网络不推荐用于于生产环境。


1. 启动两个 `alpine` 容器，并运行 `ash` 命令，（`-dit` 选项使得容器带着 TTY（t）以可交互的模式（i）在后台（t）运行）

```shell
$ docker run -dit --name alpine1 alpine ash

$ docker run -dit --name alpine2 alpine ash
```

2. 确认容器运行起来后探查 `bridge` 网络的信息

```shell
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "ad75fa41c76af8f9fbbefb5000c347d2cebe56b0b31a730005702b42b0c00fa7",
        "Created": "2019-03-02T04:57:51.222477618Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "03c4e0a99076a641ad4ce36b69621224d4aa974f008c9b3dd4b2cc587a9bdab6": {
                "Name": "alpine1",
                "EndpointID": "b5b1c1020c915d2e38e824e00a465c9ecd0396e0c7e6441871a8a17cd042bcf7",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "d1acd87c62615acf948153b79718ecc0f4326cfff6d55574cdcf2adef7e1e182": {
                "Name": "alpine2",
                "EndpointID": "b84c1b868e29e94ab7a7f227b58a3e47f4ea11300b8bffe8e41d2d354f4b574b",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

可以看到，这个网络中目前有 `alpine1` 和 `alpine2` 两个容器，`alpine1` 分配到的 IP 地址为 `172.17.0.2/16`，而 `alpine2` 分配到的 IP 地址为 `172.17.0.3/16`。

3. 进入 `alpine1`

```shell
$ docker attach alpine1

/ #
```

4. 查看 `alpine1` 的网络设备

```shell
/ # ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN qlen 1
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
205: eth0@if206: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

5. 尝试与 `alpine2` 通信

```shell
/ # ping -c 2 172.17.0.3
PING 172.17.0.3 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.141 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.217 ms
```

#### 使用用户定义的 bridge 网络

1. 先创建一个名为 `alpine-net` 的 bridge 网络

```shell
$ docker network create --driver bridge alpine-net
15e45a28a73cdbe96cfd3a32f0c6329871a70ced29ab7d8175a54059130d6488
```

2. 确认该网络的状态

```shell
$ docker network inspect alpine-net
[
    {
        "Name": "alpine-net",
        "Id": "15e45a28a73cdbe96cfd3a32f0c6329871a70ced29ab7d8175a54059130d6488",
        "Created": "2019-04-01T15:20:45.4514219Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.29.0.0/16",
                    "Gateway": "172.29.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

3. 创建四个容器，将 `alpine1`，`alpine2` 及 `alpine4` 加入 `alpine-net` 网络中，将 `alpine3` 和 `alpine4` 加入默认的 bridge 网络中（由于 `docker run` 命令中的 `--network` 选项一次只能连接一个网络，所以 `alpine4` 在创建好后需要通过 `docker network connect` 来加入默认 bridge 网络）

```shell
$ docker run -dit --name alpine1 --network alpine-net alpine ash
645e623f184f643e3e2f9aa953a020c10df11bbfad47a15cda81a093e4019370

$ docker run -dit --name alpine2 --network alpine-net alpine ash
4843f601f9787d5af7453874e1645b1947fd20bd40679a46e2be9e29511e6664

$ docker run -dit --name alpine3 alpine ash
3d738a2a015368b4bcf427eaee052995b0e1b024a5fa17dc3fc764ca7eb9b5b8

$ docker run -dit --name alpine4 --network alpine-net alpine ash
6717b0b2a9acff2635a1aff411e99dfaaa0425d5873fb08ca1df11bc2ca07278

$ docker network connect bridge alpine4
```

4. 确认 `alpine-net` 及默认 bridge 网络的状态

```shell
$ docker network inspect alpine-net
[
    {
        "Name": "alpine-net",
        "Id": "15e45a28a73cdbe96cfd3a32f0c6329871a70ced29ab7d8175a54059130d6488",
        "Created": "2019-04-01T15:20:45.4514219Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.29.0.0/16",
                    "Gateway": "172.29.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "4843f601f9787d5af7453874e1645b1947fd20bd40679a46e2be9e29511e6664": {
                "Name": "alpine2",
                "EndpointID": "ca50f28a382521c055db93bc2e8e9717a0338947e2638362e79cf215d05be631",
                "MacAddress": "02:42:ac:1d:00:03",
                "IPv4Address": "172.29.0.3/16",
                "IPv6Address": ""
            },
            "645e623f184f643e3e2f9aa953a020c10df11bbfad47a15cda81a093e4019370": {
                "Name": "alpine1",
                "EndpointID": "d8933f0ac8c55ed582de4fd8a03abe03872f87f487b1ded7a2788ba68a8ca266",
                "MacAddress": "02:42:ac:1d:00:02",
                "IPv4Address": "172.29.0.2/16",
                "IPv6Address": ""
            },
            "6717b0b2a9acff2635a1aff411e99dfaaa0425d5873fb08ca1df11bc2ca07278": {
                "Name": "alpine4",
                "EndpointID": "a2172fb70339c71ff02ce5b4c955665a186582207ad6e59b7535607098850cfa",
                "MacAddress": "02:42:ac:1d:00:04",
                "IPv4Address": "172.29.0.4/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]

$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "ad75fa41c76af8f9fbbefb5000c347d2cebe56b0b31a730005702b42b0c00fa7",
        "Created": "2019-03-02T04:57:51.222477618Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "3d738a2a015368b4bcf427eaee052995b0e1b024a5fa17dc3fc764ca7eb9b5b8": {
                "Name": "alpine3",
                "EndpointID": "1483be13d35e9ef035d866b145d853a931fb8f9aae7f2e66003bce6f0f4fe3a1",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "6717b0b2a9acff2635a1aff411e99dfaaa0425d5873fb08ca1df11bc2ca07278": {
                "Name": "alpine4",
                "EndpointID": "584ae19dbe92beb6fb002b8aae909eea6ca866a9a41b3976afca86fdaacfa081",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

5. 在用户定义的网络中，容器之间不仅能够根据彼此的 IP 地址进行通信，还能够根据容器名来进行通信，这个特性被称为**服务发现**，进入 `alpine1` 容器中，并尝试使用容器名与其他容器进行通信，由于 `alpine1` 与 `alpine3` 不在同一个网络中，所以不管通过容器名还是 IP 地址都无法与其进行通信

```shell
$ docker container attach alpine1
/ # ping -c 2 alpine2
PING alpine2 (172.29.0.3): 56 data bytes
64 bytes from 172.29.0.3: seq=0 ttl=64 time=0.296 ms
64 bytes from 172.29.0.3: seq=1 ttl=64 time=0.211 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.211/0.253/0.296 ms
/ # ping -c 2 alpine1
PING alpine1 (172.29.0.2): 56 data bytes
64 bytes from 172.29.0.2: seq=0 ttl=64 time=0.064 ms
64 bytes from 172.29.0.2: seq=1 ttl=64 time=0.160 ms

--- alpine1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.064/0.112/0.160 ms
/ # ping -c 2 alpine4
PING alpine4 (172.29.0.4): 56 data bytes
64 bytes from 172.29.0.4: seq=0 ttl=64 time=0.428 ms
64 bytes from 172.29.0.4: seq=1 ttl=64 time=0.190 ms

--- alpine4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.190/0.309/0.428 ms
/ # ping -c 2 alpine3
ping: bad address 'alpine3'
/ # ping -c 2 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes
^C
--- 172.17.0.2 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```

6. 进入 `alpine4`，尝试与其他容器通信

```shell
$ docker container attach alpine4
/ # ping -c 2 alpine1
PING alpine1 (172.29.0.2): 56 data bytes
64 bytes from 172.29.0.2: seq=0 ttl=64 time=0.246 ms
64 bytes from 172.29.0.2: seq=1 ttl=64 time=0.241 ms

--- alpine1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.241/0.243/0.246 ms
/ # ping -c 2 alpine2
PING alpine2 (172.29.0.3): 56 data bytes
64 bytes from 172.29.0.3: seq=0 ttl=64 time=0.281 ms
64 bytes from 172.29.0.3: seq=1 ttl=64 time=0.194 ms

--- alpine2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.194/0.237/0.281 ms
/ # ping -c 2 alpine3
ping: bad address 'alpine3'
/ # ping -c 2 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.152 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.215 ms

--- 172.17.0.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.152/0.183/0.215 ms
```

## 使用 host 网络

如果某个容器使用的是 host 网络，那么该容器的网络栈是不会与 Docker 宿主机进行隔离的。举个例子，如果某个容器使用 host 网络并绑定了 80 端口，那么该绑定该端口的应用将能够在宿主机 IP 的 80 端口上访问到。

**要注意的是，host 类型的驱动只在 Linux 宿主机下起作用。**


### host 网络实践

本次实践会运行一个 `nginx` 容器，并将其直接绑定到宿主机的 80 端口上。从网络的层次上看，容器与宿主机的隔离程度和直接在宿主机上运行 `nginx` 进程是一样的。但是从存储，进程命名空间和用户命名空间上看，`nginx` 进程与宿主机是完全隔离的。

1. 启动 `nginx` 容器（`--rm` 选项会移除已有的容器）

```shell
$ docker run --rm -d --network host --name my_nginx nginx
```

2. 访问 http://localhost:80/

## 使用 overlay 网络

overlay 网络驱动会在多个 Docker 宿主机之间建立一个分布式网络。该网络位于宿主机的网络之上，所以被称为 overlay 网络，容器能够加入到该网络中，并与该网络中的其他容器安全地进行通讯。

当你初始化一个 swarm 节点并将其加入一个已经存在的 swarm 中时，该节点的宿主机上会创建两个新的网络：

- 一个名为 `ingress` 的 overlay 网络，负责处理 swarm service 之间的通信数据。当你创建的 swarm service 没有加入自定义的 overlay 网络中时，其默认会加入 `ingress` 中。
- 一个名为 `docekr_gwbridge` 的 bridge 网络，其负责将 swarm 中的 Docker 进程连接起来。

通过 `docker network create` 命令能够创建自定义的 overlay 网络，这与创建自定义的 bridge 网络相似。一个 swarm service 或容器能够同时加入多个网络中。


## 使用 macvlan 网络

macvlan 网络能够为容器分配一个 MAC 地址，使得容器看起来就像是物理网络中的一台真实物理设备一样。但是，这需要为 macvlan 网络分配一个 Docker 宿主机上的物理网络接口，该接口会用于该网络及其子网和网关。你甚至能够使用不同的物理接口来对不同的 macvlan 网络进行隔离，但是在使用 macvlan 网络前，必须要明确以下几点：

- 很可能会因为 IP 地址耗尽或是 VLAN spread 而无意中损坏你的网络。
- 你的网络设备需要能够处理混杂模式，即在一个物理接口上分配多个 MAC 地址。
- 如果你的应用在 bridge 网络或是 overlay 网络上也能够正常工作，那么从长远来看这些都是更好的解决方案。

macvlan 网络存在两种模式：bridge 模式和 802.1q trun bridge 模式，这两种模式的差异如下：

- 在 bridge 模式下，macvlan 网络的流量会直接经过宿主机上的网络接口。
- 在 802.1q trun bridge 模式下，网络流量会经过一个 Docker 创建的 802.1q 子接口。这个模式允许我们更加精细对流量进行路由和过滤。
