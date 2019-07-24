## 一 consul介绍 

Consul是HashiCorp公司推出的开源工具，用于实现分布式系统的服务发现与配置。 Consul是分布式、高可用、可横向扩展的。  

它具备以下特性：
- service discovery：consul通过DNS或者HTTP接口使服务注册和服务发现变的很容易，一些外部服务，例如saas提供的也可以一样注册。
- health checking：健康检测使consul可以快速的对集群中的操作发出警报。和服务发现的集成，可以防止服务转发到 故障的服务上面。
- key/value storage：一个用来存储动态配置的系统。提供简单的HTTP接口，可以在任何地方操作。
- multi-datacenter：无需复杂的配置，即可支持任意数量的区域。  

## 二 consul安装

下载地址：https://www.consul.io/downloads.html

mac和linux版只需要将二进制文件移动到 `/usr/local/bin`目录中即可，测试安装结果：
```
consul
```

## 三 consul常识

#### 3.1 Consul 的角色

Consul是典型的 C/S 架构，必须运行agent，agent可以运行为server或client模式可：
- client: 无状态客户端，用来将 HTTP 和 DNS 接口请求转发给局域网内的服务端集群.
- server: 高可用集群服务端，用来保存配置信息，在局域网内与本地客户端通讯，通过广域网与其他数据中心通讯。每个数据中心的 server 数量推荐为 3/5 个。

每个数据中心至少必须拥有一台server，建议在一个集群中有3或者5个server，部署单一的server，在出现失败时会不可避免的造成数据丢失。  

其他的agent运行为client模式，一个client是一个非常轻量级的进程，用于注册服务。

#### 3.2 启动 Consul Server

服务端节点启动命令：
```
# 以server模式运行agent，启动节点node1
consul agent -server -bootstrap-expect 2 -data-dir ~/tmp/consul/data -node=n1 -bind=192.168.110.123 -ui -config-dir /etc/consul.d -rejoin -join 192.168.110.123 -client 0.0.0.0

# 以server模式运行agent，启动节点node2
consul agent -server -bootstrap-expect 2 -data-dir ~/tmp/consul/data -node=n2 -bind=192.168.110.148 -ui -rejoin -join 192.168.110.123
```

参数解释：
- server：定义agent运行在server模式
- bootstrap-expect：期望提供的server节点数，提供该值后，consul一直等到sever数目达到该值时才会引导整个集群，注意不能和bootstrap共用 
- data-dir：存放agent状态的目录
- node：节点在集群中的名称，必须是唯一的，默认是该节点的主机名 
- bind：该地址用来在集群内部的通讯，集群内的所有节点到地址都必须是可达的，默认是0.0.0.0
- ui：启动web界面
- config-dir：配置文件目录，里面所有以.json结尾的文件都会被加载 -rejoin:使consul忽略先前的离开，在再次启动后仍旧尝试加入集群中。
- client：consul服务侦听地址，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1所以不对外提供服 务，如果你要对外提供服务改成0.0.0.0
- rejoin：使consul忽略先前的离开，在再次启动后仍旧尝试加入集群中。 
- config-dir：配置文件目录，里面所有以.json结尾的文件都会被加载 
- client：consul服务侦听地址，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1所以不对外提供服务，如果你要对外提供服务改成0.0.0.0
- join：启动时加入这个哪个集群


#### 3.3 启动Consul Client

node3：
```
# 以client模式运行cosnul agent，-join 加入到已有的集群中去。
consul agent -data-dir /tmp/consul -node=n3 -bind=192.168.110.124 -config-dir /etc/consul.d -rejoin -join 192.168.110.123
```

#### 3.4 查看集群成员

新开一个终端窗口运行consul members, 你可以看到Consul集群的成员:
```
consul members
```

#### 3.5 停止Agent

使用Ctrl+C可以关闭Agent，在退出中，Consul会提醒其他集群成员，这个节点离开了。如果强行杀掉进程，则集群的其他成员应该能检测到这个节点失效：
- 当一个成员离开，他的服务和检测也会从目录中移除
- 当一个成员失效了，他的健康状况被简单的标记为危险，但是不会从目录中移除
- Consul会自动尝试对失效的节点进行重连

但是一般我们需要优雅关闭，可以避免引起潜在的可用性故障影响达成一致性协议：
```
consul leave
```

## 四 Consul中常见术语

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

## 四 consul架构 

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