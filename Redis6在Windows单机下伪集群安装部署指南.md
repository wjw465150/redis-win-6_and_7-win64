# Redis6在Windows单机下伪集群安装部署指南

## 前言

redis集群化部署主要用于大型缓存架构.一般的小型架构，使用redis主从配置就行.

使用redis集群可以方便快捷地对集群进行动态扩容，动态的添加、删除节点，reshard、并带有自动故障恢复功能。

一般redis集群使用3主3从，并且尽量保证主服务器与从服务器不在同一台机器上，防止机器故障导致的集群瘫痪，每个主 服务器搭配一个从服务器，保证集群的高可用性。

 **软件版本：**

- OS：Windows10-64位

- Redis：redis-win-6.2.6-win64.zip(下载地址`https://github.com/wjw465150/redis-win-6-win64`)

# Redis配置

假设Redis解压在`D:\redis-win-6.2.6-win64`目录下.建立配置Redis集群时会用到的文件夹：

```bash
D:\redis-win-6.2.6-win64\conf
D:\redis-win-6.2.6-win64\log
D:\redis-win-6.2.6-win64\run

D:\redis-win-6.2.6-win64\data\7001
D:\redis-win-6.2.6-win64\data\7002
D:\redis-win-6.2.6-win64\data\7003
D:\redis-win-6.2.6-win64\data\8001
D:\redis-win-6.2.6-win64\data\8002
D:\redis-win-6.2.6-win64\data\8003
```

**创建redis配置文件:**  `D:\redis-win-6.2.6-win64\conf\redis_7001.conf`

打开`redis_7001.conf`文件，修改成以下内容：

```ini
#添加本机的ip
bind 0.0.0.0
#端口
port 7001

#AOF持久化
save  #save后给空值，表示禁用RDB持久化策略
appendonly yes
#守护进程
daemonize yes

#pid存储目录
pidfile ./run/redis_7001.pid
#日志存储目录
logfile ./log/redis_7001.log
#数据存储目录，目录要提前创建好
dir ./data/7001

#开启集群
cluster-enabled yes
#集群节点配置文件，这个文件是不能手动编辑的。确保每一个集群节点的配置文件不同
cluster-config-file nodes_7001.conf
#集群节点的超时时间，单位：ms，超时后集群会认为该节点失败
cluster-node-timeout 15000

#->@wjw_note: 一下是根据实际情况来填写
# Close the connection after a client is idle for N seconds (0 to disable)
timeout 0
loglevel warning
databases 16
# 设置Redis占用内存的大小
maxmemory 100mb

# 如果内存满了就需要按照如相应算法进行删除过期的/最老的
# volatile-lru 根据LRU算法移除设置了过期的key
# allkeys-lru  根据LRU算法移除任何key(包含那些未设置过期时间的key)
# volatile-random/allkeys->random 使用随机算法而不是LRU进行删除
# volatile-ttl 根据Time-To-Live移除即将过期的key
# noeviction   永不过期,而是报错
maxmemory-policy volatile-lru
# Redis并不是真正的LRU/TTL,而是基于采样进行移除的,即如采样10个数据移除其中最老的/即将过期的
maxmemory-samples 10
# 持久化策略,默认每秒fsync一次,也可以选择always即每次操作都进行持久化,或者no表示不进行持久化而是借助操作系统的同步将缓存区数据写到磁盘
appendfsync everysec

# AOF重写策略(同时满足如下两个策略进行重写)
# 当AOF文件大小占到初始文件大小的多少百分比时进行重写
auto-aof-rewrite-percentage 100
# 触发重写的最小文件大小
auto-aof-rewrite-min-size 1gb

# 为减少磁盘操作,暂缓重写阶段的磁盘同步
no-appendfsync-on-rewrite yes

# 慢查
# 下面的时间单位是微秒,所以1000000就是1秒.注意,负数时间会禁用慢查询日志,而0则会强制记录所有命令.
slowlog-log-slower-than 10000
# 这个长度没有限制.只要有足够的内存就行.你可以通过 SLOWLOG RESET 来释放内存.(注:慢查日志是在内存里)
slowlog-max-len 128
#<-@wjw_note: 一下是根据实际情况来填写
```

然后把`redis_7001.conf`复制5份: `redis_7002.conf`,`redis_7003.conf`,`redis_8001.conf`,`redis_8002.conf`,`redis_8003.conf`.

**修改这4处地方:**

- #pid存储目录 `pidfile ./run/redis_7001.pid`
- #日志存储目录: `logfile ./log/redis_7001.log`
- #数据存储目录: `dir ./data/7001`
- #集群节点配置文件: `cluster-config-file nodes_7001.conf`

> **⚠重要:**  把文件里的端口号`7001`修改成各自的`7002,7003,8001,8002,8003`



**制作启动配置文件**

创建启动脚本：`cluster_start.bat`

```bash
REM 进入当前批处理文件所在的目录
cd /d %~dp0

redis-server.exe ./conf/redis_7001.conf
redis-server.exe ./conf/redis_8001.conf

redis-server.exe ./conf/redis_7002.conf
redis-server.exe ./conf/redis_8002.conf

redis-server.exe ./conf/redis_7003.conf
redis-server.exe ./conf/redis_8003.conf
```



创建关闭脚本：`cluster_shutdown.bat`

```bash
REM 进入当前批处理文件所在的目录
cd /d %~dp0

redis-cli.exe -h localhost -p 7001 shutdown
redis-cli.exe -h localhost -p 8001 shutdown

redis-cli.exe -h localhost -p 7002 shutdown
redis-cli.exe -h localhost -p 8002 shutdown

redis-cli.exe -h localhost -p 7003 shutdown
redis-cli.exe -h localhost -p 8003 shutdown
```

# Redis集群

**1、创建集群**

用以下命令创建集群，`--cluster-replicas 1` 参数表示希望每个主服务器都有一个从服务器，这里则代表3主3从，前3个代表3个master，后3个代表3个slave。

输入: 
```bash
redis-cli.exe --cluster create 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:8001 127.0.0.1:8002 127.0.0.1:8003 --cluster-replicas 1
```

>  通过该方式创建的带有从节点的机器不能够自己手动指定主节点，redis集群会尽量把主从服务器分配在不同机 器上。

输出如下:

```
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:8002 to 127.0.0.1:7001
Adding replica 127.0.0.1:8003 to 127.0.0.1:7002
Adding replica 127.0.0.1:8001 to 127.0.0.1:7003
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 75bbb16501a9ea766a35463b0a2a19fdb58f1ab3 127.0.0.1:7001
   slots:[0-5460] (5461 slots) master
M: 15f947ab6196a0c770f57b66e6fe82fa1546e5ae 127.0.0.1:7002
   slots:[5461-10922] (5462 slots) master
M: fa1df2b31ab26d65ac61a3d7c26dd9555ced394c 127.0.0.1:7003
   slots:[10923-16383] (5461 slots) master
S: 12b22cafc02d9e460007042e9cd7f600185cf68c 127.0.0.1:8001
   replicates 75bbb16501a9ea766a35463b0a2a19fdb58f1ab3
S: 473f8fd14ba88e4d90a692f074809179b6baa487 127.0.0.1:8002
   replicates 15f947ab6196a0c770f57b66e6fe82fa1546e5ae
S: 7c539803fa16f9303b1c158ee216bf539cf9e626 127.0.0.1:8003
   replicates fa1df2b31ab26d65ac61a3d7c26dd9555ced394c
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 127.0.0.1:7001)
M: 75bbb16501a9ea766a35463b0a2a19fdb58f1ab3 127.0.0.1:7001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 15f947ab6196a0c770f57b66e6fe82fa1546e5ae 127.0.0.1:7002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 473f8fd14ba88e4d90a692f074809179b6baa487 127.0.0.1:8002
   slots: (0 slots) slave
   replicates 15f947ab6196a0c770f57b66e6fe82fa1546e5ae
S: 7c539803fa16f9303b1c158ee216bf539cf9e626 127.0.0.1:8003
   slots: (0 slots) slave
   replicates fa1df2b31ab26d65ac61a3d7c26dd9555ced394c
S: 12b22cafc02d9e460007042e9cd7f600185cf68c 127.0.0.1:8001
   slots: (0 slots) slave
   replicates 75bbb16501a9ea766a35463b0a2a19fdb58f1ab3
M: fa1df2b31ab26d65ac61a3d7c26dd9555ced394c 127.0.0.1:7003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

> **🏷注意:** 注意看M和S，对照下集群角色表

**2、查看集群状态**

输入: `redis-cli -c -h 127.0.0.1 -p 7001 cluster info`

输出:

```
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:170
cluster_stats_messages_pong_sent:168
cluster_stats_messages_sent:338
cluster_stats_messages_ping_received:163
cluster_stats_messages_pong_received:170
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:338
```

**3、查看集群节点**

输入: `redis-cli -c -h 127.0.0.1 -p 7001 cluster nodes`

输出:

```
75bbb16501a9ea766a35463b0a2a19fdb58f1ab3 127.0.0.1:7001@17001 myself,master - 0 1641630845000 1 connected 0-5460
15f947ab6196a0c770f57b66e6fe82fa1546e5ae 127.0.0.1:7002@17002 master - 0 1641630846251 2 connected 5461-10922
473f8fd14ba88e4d90a692f074809179b6baa487 127.0.0.1:8002@18002 slave 15f947ab6196a0c770f57b66e6fe82fa1546e5ae 0 1641630844063 2 connected
7c539803fa16f9303b1c158ee216bf539cf9e626 127.0.0.1:8003@18003 slave fa1df2b31ab26d65ac61a3d7c26dd9555ced394c 0 1641630845154 3 connected
12b22cafc02d9e460007042e9cd7f600185cf68c 127.0.0.1:8001@18001 slave 75bbb16501a9ea766a35463b0a2a19fdb58f1ab3 0 1641630847344 1 connected
fa1df2b31ab26d65ac61a3d7c26dd9555ced394c 127.0.0.1:7003@17003 master - 0 1641630844000 3 connected 10923-16383
```

# 测试用例

```
# redis-cli -c -h 127.0.0.1 -p 7001
127.0.0.1:7001> set name node1
-> Redirected to slot [5798] located at 127.0.0.1:7002
OK

# redis-cli -c -h 127.0.0.1 -p 7002
127.0.0.1:7002> get name
"node1"
```

