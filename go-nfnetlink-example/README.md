# 使用 Go 对接 iptables NFQUEUE 的例子

最近在学习 `iptables NFQUEUE` 的时候，顺手使用 Go 语言写了一个例子。

> 源代码：[github.com/Asphaltt/go-nfnetlink-example](https://github.com/Asphaltt/go-nfnetlink-example)

## 例子的效果

使用 `iptables NFQUEUE` 监听新建 tcp 连接：

```bash
# iptables -t raw -I PREROUTING -p tcp --syn -j NFQUEUE --queue-num=1 --queue-bypass
# go-nfnetlink-example
A new tcp connection will be established: 187.10.95.158:55944 -> 10.0.2.1:22
A new tcp connection will be established: 185.246.130.20:7308 -> 10.0.2.1:22
A new tcp connection will be established: 89.248.165.22:50338 -> 10.0.2.1:34308
A new tcp connection will be established: 119.61.0.141:36558 -> 10.0.2.1:28050
A new tcp connection will be established: 89.248.165.186:51093 -> 10.0.2.1:25550
A new tcp connection will be established: 164.92.170.186:51946 -> 10.0.2.1:22
A new tcp connection will be established: 89.248.165.194:51879 -> 10.0.2.1:45063
A new tcp connection will be established: 152.89.198.183:44922 -> 10.0.2.1:17423
A new tcp connection will be established: 185.246.130.20:48382 -> 10.0.2.1:22
A new tcp connection will be established: 157.230.9.57:56656 -> 10.0.2.1:22
A new tcp connection will be established: 185.246.130.20:33961 -> 10.0.2.1:22
A new tcp connection will be established: 103.149.196.186:60710 -> 10.0.2.1:22
A new tcp connection will be established: 185.246.130.20:6985 -> 10.0.2.1:22
A new tcp connection will be established: 193.47.61.208:59731 -> 10.0.2.1:52002
A new tcp connection will be established: 111.7.96.133:32587 -> 10.0.2.1:41579
A new tcp connection will be established: 190.128.239.54:47034 -> 10.0.2.1:22
A new tcp connection will be established: 206.189.142.4:59694 -> 10.0.2.1:24439
A new tcp connection will be established: 185.246.130.20:34919 -> 10.0.2.1:22
A new tcp connection will be established: 89.248.165.193:55705 -> 10.0.2.1:10351
A new tcp connection will be established: 152.89.198.175:57939 -> 10.0.2.1:17681
A new tcp connection will be established: 74.82.47.11:57278 -> 10.0.2.1:44818
A new tcp connection will be established: 62.204.41.206:57984 -> 10.0.2.1:18074
A new tcp connection will be established: 176.111.174.100:56835 -> 10.0.2.1:17805
A new tcp connection will be established: 164.92.170.186:55342 -> 10.0.2.1:22
A new tcp connection will be established: 185.246.130.20:1028 -> 10.0.2.1:22
```

## 简洁的代码

使用了 [go-nfnetlink](https://github.com/subgraph/go-nfnetlink) 纯 Go 实现的 `nfnetlink` 库，不依赖 `libnetfilter_queue` Linux 系统库，编译后即可使用。

从 `iptables NFQUEUE` 中接收 tcp 连接的 **SYN** 包，并从 **SYN** 包中解析得到源 IP 地址、源 tcp 端口、目的 IP 地址、目的 tcp 端口等信息。

```go
	q := nfqueue.NewNFQueue(1)

	ps, err := q.Open()
	if err != nil {
		fmt.Printf("Error opening NFQueue: %v\n", err)
		os.Exit(1)
	}
	defer q.Close()

	for p := range ps {
		networkLayer := p.Packet.NetworkLayer()
		ipsrc, ipdst := networkLayer.NetworkFlow().Endpoints()

		transportLayer := p.Packet.TransportLayer()
		tcpsrc, tcpdst := transportLayer.TransportFlow().Endpoints()

		fmt.Printf("A new tcp connection will be established: %s:%s -> %s:%s\n",
			ipsrc, tcpsrc, ipdst, tcpdst)
		p.Accept()
	}
```

使用的 `iptables` 规则如下：

```bash
iptables -t raw -I PREROUTING -p tcp --syn -j NFQUEUE --queue-num=1 --queue-bypass
```

在 `raw` 表 `PREROUTING` 链上匹配 tcp 连接的 **SYN** 包。

## `iptables NFQUEUE`

> ### NFQUEUE
>
> This target passes the packet to userspace using the **nfnetlink_queue** handler. The packet is put into the queue identified by its 16-bit queue number. Userspace can inspect and modify the packet if desired. Userspace must then drop or reinject the packet into the kernel. Please see libnetfilter_queue for details. **nfnetlink_queue** was added in Linux 2.6.14. The **queue-balance** option was added in Linux 2.6.31, **queue-bypass** in 2.6.39.
>
> - **--queue-num** *value*
>
>   This specifies the QUEUE number to use. Valid queue numbers are 0 to 65535. The default value is 0.
>
>
>
> - **--queue-balance** *value*:*value*
>
>   This specifies a range of queues to use. Packets are then balanced across the given queues. This is useful for multicore systems: start multiple instances of the userspace program on queues x, x+1, .. x+n and use "--queue-balance *x*:*x+n*". Packets belonging to the same connection are put into the same nfqueue.
>
>
>
> - **--queue-bypass**
>
>   By default, if no userspace program is listening on an NFQUEUE, then all packets that are to be queued are dropped. When this option is used, the NFQUEUE rule behaves like ACCEPT instead, and the packet will move on to the next table.
>
>
>
> - **--queue-cpu-fanout**
>
>   Available starting Linux kernel 3.10. When used together with **--queue-balance** this will use the CPU ID as an index to map packets to the queues. The idea is that you can improve performance if there's a queue per CPU. This requires **--queue-balance** to be specified.
>
> Doc: [iptables-extensions](https://ipset.netfilter.org/iptables-extensions.man.html)
