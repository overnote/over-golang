## 一 consul集群设置

#### 1.1 集群启动 Consul Server

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


#### 1.2 集群启动Consul Client

node3：
```
# 以client模式运行cosnul agent，-join 加入到已有的集群中去。
consul agent -data-dir /tmp/consul -node=n3 -bind=192.168.110.124 -config-dir /etc/consul.d -rejoin -join 192.168.110.123
```

#### 1.3 查看集群成员

新开一个终端窗口运行consul members, 你可以看到Consul集群的成员:
```
consul members
```

#### 1.4 停止Agent

使用Ctrl+C可以关闭Agent，在退出中，Consul会提醒其他集群成员，这个节点离开了。如果强行杀掉进程，则集群的其他成员应该能检测到这个节点失效：
- 当一个成员离开，他的服务和检测也会从目录中移除
- 当一个成员失效了，他的健康状况被简单的标记为危险，但是不会从目录中移除
- Consul会自动尝试对失效的节点进行重连

但是一般我们需要优雅关闭，可以避免引起潜在的可用性故障影响达成一致性协议：
```
consul leave
```

## 贴士

较全的consul介绍：https://blog.csdn.net/liuzhuchen/article/details/81913562