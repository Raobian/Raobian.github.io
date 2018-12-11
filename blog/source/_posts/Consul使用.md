---
title: Consul使用
date: 2018-12-11 17:07:18
tags: Consul
categories: utils
---



### 1. 环境
node1 192.168.1.101  作为client
node2 192.168.1.102  作为client
node3 192.168.1.103  作为server

### 2. 安装
下载指定版本
```
https://www.consul.io/downloads.html
```
解压，拷贝到/usr/bin

```
cp consule /usr/bin/
```
<!-- more -->
### 3. 创建目录

```
/opt/bian/consul.d/config.json      json格式配置文件  
/opt/bian/consul_data               data路径
```

### 4. Server配置与启动
##### 4.1. 配置文件
路径 /opt/bian/consul.d/config.json
```
{
    "bootstrap_expect": 1,
    "server": true,
    "datacenter": "Bian",
    "data_dir": "/opt/bian/consul_data",
    "log_level": "INFO",
    "enable_syslog": true
}
```
##### 4.2 启动
前台运行，建议screen
```
consul agent -config-dir /opt/bian/consul.d/ -bind=192.168.1.103 -ui -client 0.0.0.0
```

### 5. Client配置与启动
##### 5.1 配置文件
路径 /opt/bian/consul.d/config.json
```
{
    "server": false,
    "datacenter": "Bian",
    "data_dir": "/opt/bian/consul_data",
    "bind_addr": "192.168.1.102",
    "log_level": "INFO",
    "enable_syslog": true
}
```

##### 5.2 启动
```
consul agent -config-dir /opt/bian/consul.d/ --retry-join 192.168.1.103
```

### 6. 使用
##### 6.1 members， info， join， leave， keygen， reload
reload 只支持部分参数，不可从client变为server
##### 6.2 kv
```
    delete    Removes data from the KV store
    export    Exports a tree from the KV store as JSON
    get       Retrieves or lists data from the KV store
    import    Imports a tree stored as JSON to the KV store
    put       Sets or updates data in the KV store

```


### 7. 参数说明
##### 7.1 执行参数
```
-advertise：通知展现地址用来改变我们给集群中的其他节点展现的地址，一般情况下-bind地址就是展现地址
-bootstrap：用来控制一个server是否在bootstrap模式，在一个datacenter中只能有一个server处于bootstrap模式，当一个server处于bootstrap模式时，可以自己选举为raft leader。
-bootstrap-expect：在一个datacenter中期望提供的server节点数目，当该值提供的时候，consul一直等到达到指定sever数目的时候才会引导整个集群，该标记不能和bootstrap公用
-bind：该地址用来在集群内部的通讯，集群内的所有节点到地址都必须是可达的，默认是0.0.0.0
-client：consul绑定在哪个client地址上，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1
-config-file：明确的指定要加载哪个配置文件
-config-dir：配置文件目录，里面所有以.json结尾的文件都会被加载
-data-dir：提供一个目录用来存放agent的状态，所有的agent允许都需要该目录，该目录必须是稳定的，系统重启后都继续存在
-dc：该标记控制agent允许的datacenter的名称，默认是dc1
-encrypt：指定secret key，使consul在通讯时进行加密，key可以通过consul keygen生成，同一个集群中的节点必须使用相同的key
-join：加入一个已经启动的agent的ip地址，可以多次指定多个agent的地址。如果consul不能加入任何指定的地址中，则agent会启动失败，默认agent启动时不会加入任何节点。
-retry-join：和join类似，但是允许你在第一次失败后进行尝试。
-retry-interval：两次join之间的时间间隔，默认是30s
-retry-max：尝试重复join的次数，默认是0，也就是无限次尝试
-log-level：consul agent启动后显示的日志信息级别。默认是info，可选：trace、debug、info、warn、err。
-node：节点在集群中的名称，在一个集群中必须是唯一的，默认是该节点的主机名
-protocol：consul使用的协议版本
-rejoin：使consul忽略先前的离开，在再次启动后仍旧尝试加入集群中。
-server：定义agent运行在server模式，每个集群至少有一个server，建议每个集群的server不要超过5个
-syslog：开启系统日志功能，只在linux/osx上生效
-ui-dir:提供存放web ui资源的路径，该目录必须是可读的
-pid-file:提供一个路径来存放pid文件，可以使用该文件进行SIGINT/SIGHUP(关闭/更新)agent
```

##### 7.2 配置参数
```
acl_datacenter：只用于server，指定的datacenter的权威ACL信息，所有的servers和datacenter必须同意ACL datacenter
acl_default_policy：默认是allow
acl_down_policy：
acl_master_token：
acl_token：agent会使用这个token和consul server进行请求
acl_ttl：控制TTL的cache，默认是30s
addresses：一个嵌套对象，可以设置以下key：dns、http、rpc
advertise_addr：等同于-advertise
bootstrap：等同于-bootstrap
bootstrap_expect：等同于-bootstrap-expect
bind_addr：等同于-bind
ca_file：提供CA文件路径，用来检查客户端或者服务端的链接
cert_file：必须和key_file一起
check_update_interval：
client_addr：等同于-client
datacenter：等同于-dc
data_dir：等同于-data-dir
disable_anonymous_signature：在进行更新检查时禁止匿名签名
disable_remote_exec：禁止支持远程执行，设置为true，agent会忽视所有进入的远程执行请求
disable_update_check：禁止自动检查安全公告和新版本信息
dns_config：是一个嵌套对象，可以设置以下参数：allow_stale、max_stale、node_ttl 、service_ttl、enable_truncate
domain：默认情况下consul在进行DNS查询时，查询的是consul域，可以通过该参数进行修改
enable_debug：开启debug模式
enable_syslog：等同于-syslog
encrypt：等同于-encrypt
key_file：提供私钥的路径
leave_on_terminate：默认是false，如果为true，当agent收到一个TERM信号的时候，它会发送leave信息到集群中的其他节点上。
log_level：等同于-log-level
node_name:等同于-node
ports：这是一个嵌套对象，可以设置以下key：dns(dns地址：8600)、http(http api地址：8500)、rpc(rpc:8400)、serf_lan(lan port:8301)、serf_wan(wan port:8302)、server(server rpc:8300)
protocol：等同于-protocol
recursor：
rejoin_after_leave：等同于-rejoin
retry_join：等同于-retry-join
retry_interval：等同于-retry-interval
server：等同于-server
server_name：会覆盖TLS CA的node_name，可以用来确认CA name和hostname相匹配
skip_leave_on_interrupt：和leave_on_terminate比较类似，不过只影响当前句柄
start_join：一个字符数组提供的节点地址会在启动时被加入
statsd_addr：
statsite_addr：
syslog_facility：当enable_syslog被提供后，该参数控制哪个级别的信息被发送，默认Local0
ui_dir：等同于-ui-dir
verify_incoming：默认false，如果为true，则所有进入链接都需要使用TLS，需要客户端使用ca_file提供ca文件，只用于consul server端，因为client从来没有进入的链接
verify_outgoing：默认false，如果为true，则所有出去链接都需要使用TLS，需要服务端使用ca_file提供ca文件，consul server和client都需要使用，因为两者都有出去的链接
watches：watch一个详细名单
```

### 8. 服务发现 Consul-Template
与 etcd+confd类似


