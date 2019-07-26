## 一 Consul 的角色

Consul是典型的 C/S 架构，必须运行agent，agent可以运行为server或client模式可：
- client: 无状态客户端，用来将 HTTP 和 DNS 接口请求转发给局域网内的服务端集群.
- server: 高可用集群服务端，用来保存配置信息，在局域网内与本地客户端通讯，通过广域网与其他数据中心通讯。每个数据中心的 server 数量推荐为 3/5 个。

每个数据中心至少必须拥有一台server，建议在一个集群中有3或者5个server，部署单一的server，在出现失败时会不可避免的造成数据丢失。  

其他的agent运行为client模式，一个client是一个非常轻量级的进程，用于注册服务。


## 二 Consul中常见术语

```
agent		指令是consul的核心，它运行agent来维护成员的重要信息、运行检查、服务宣布、查询处理等等。

event		命令提供了一种机制，用来fire自定义的用户事件，这些事件对consul来说是不透明的，
			但它们可以用来构建自动部署、重启服务或者其他行动的脚本。

exec		指令提供了一种远程执行机制，比如你要在所有的机器上执行uptime命令，
			远程执行的工作通过job来指定，存储在KV中，agent使用event系统可以快速的知道有新的job产生，
			消息是通过gossip协议来传递的，因此消息传递是最佳的，但是并不保证命令的执行。
			事件通过gossip来驱动，远程执行依赖KV存储系统(就像消息代理一样)。

force-leave	治疗可以强制consul集群中的成员进入left状态(空闲状态)，记住，即使一个成员处于活跃状态，
			它仍旧可以再次加入集群中，这个方法的真实目的是强制移除failed的节点。如果failed的节点还是网络的一部分，
			则consul会周期性的重新链接failed的节点，如果经过一段时间后(默认是72小时)，
			consul则会宣布停止尝试链接failed的节点。force-leave指令可以快速的把failed节点转换到left状态。

info		指令提供了各种操作时可以用到的debug信息，对于client和server，info有返回不同的子系统信息，
			目前有以下几个KV信息：agent(提供agent信息)，consul(提供consul库的信息)，raft(提供raft库的信息)，
			serf_lan(提供LAN gossip pool),serf_wan(提供WAN gossip pool)

join		指令告诉consul agent加入一个已经存在的集群中，一个新的consul agent必须加入一个已经有至少一个成员的集群中，
			这样它才能加入已经存在的集群中，如果你不加入一个已经存在的集群，则agent是它自身集群的一部分，
			其他agent则可以加入进来。agents可以加入其他agent多次。consul join [options] address。
			如果你想加入多个集群，则可以写多个地址，consul会加入所有的地址。

keygen		指令生成加密的密钥，可以用在consul agent通讯加密

leave		指令触发一个优雅的离开动作并关闭agent，节点离开后不会尝试重新加入集群中。
			运行在server状态的节点，节点会被优雅的删除，这是很严重的，
			在某些情况下一个不优雅的离开会影响到集群的可用性。

members		指令输出consul agent目前所知道的所有的成员以及它们的状态，
			节点的状态只有alive、left、failed三种状态。

monitor		指令用来链接运行的agent，并显示日志。monitor会显示最近的日志，
			并持续的显示日志流，不会自动退出，除非你手动或者远程agent自己退出。
	

reload		指令可以重新加载agent的配置文件。SIGHUP指令在重新加载配置文件时使用，
			任何重新加载的错误都会写在agent的log文件中，并不会打印到屏幕。

version		打印consul的版本

watch		指令提供了一个机制，用来监视实际数据视图的改变(节点列表、成员服务、KV)，
			如果没有指定进程，当前值会被dump出来

ui 			web ui界面的命令
```

## 三 consul架构 

consul的集群是由 N 个 Server 与 M 个 Client 组成的。他们都是consul的一个节点，所有的服务都可以注册到这些节点上，正是通过这些节点实现服务注册信息的共享。  

Client 表示consul的client模式，是consul节点的一种模式，这种模式下，所有注册到当前节点的服务会被转发到 Server （通过HTTP和DNS接口请求server），本身是不持久化这些信息。     

Server 表示consul的server模式，是consul节点的一种模式，这种模式下，功能和 Client 都一样，唯一不同的是，它会把所有的信息持久化的本地，这样遇到故障信息是可以被保留的。  

ServerLeader 表明这个 Server 是它们的老大，它需要负责同步注册的信息给其它的 Server ，同时也要负责各个节点的健康监测。  

consul可以用来实现分布式系统的服务发现与配置：
- client把服务请求传递给server
- server负责提供服务以及和 其他数据中心交互。

既然server端提供了所有服务，那为何还需要多此一举地用client端来接收一 次服务请求？  

首先：server端的网络连接资源有限。对于一个分布式系统，一般情况下访问量是很大的。如果用户能不通过client直接地访问数据中心，那么数据中心必然要为每个用户提供一个单独的连接资源(线程，端口号等等)，那么server端的负担会非常大。所以很有必要用大量的client端来分散用户的连接请求，在client端先统一整合用户的服务请求，然后一次性地通过一个单一的链接发送大量的请求给 server端，以大量减少server端的网络负担。   

其次：在client端可以对用户的请求进行一些处理来提高服务的效率，比如将相同的请求合并成同一个查询，再比如将之前的查询通过cookie的形式缓存下来。但是这些功能都需要消耗不少的计算和存储资源。如果在server端提供这些功能，必然加重server端的负担，使得server端更加不稳定。而通过client端来进行这些服务就没有这些问题了，因为client端不提供实际服务，有很充足的计算资源来进行这些处理这些工作。   

最后：consul规定只要接入一个client就能将自己注册到一个服务网络当中。这种架构使得系统的可扩展性非常的强，网络的拓扑变化可以特别的灵活。这也是依赖于client—server结构的。如果系 统中只有几个数据中心存在，那网络的扩张也无从谈起了。  