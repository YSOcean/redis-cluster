
### Redis 集群模式

#### 1.为什么需要集群
　　**①、并发量**  
　　通常来说,单台Redis能够执行10万/秒的命令,这个并发基本上能够满足我们所有需求了,但有时候比如做离线计算,为了更快的得出结果,有时候我们希望超过这个并发,那这个时候单机就不满足我们需求了,就需要集群了.  
　　**②、数据量**  
    通常来说,单台服务器的内存大概在16G-256G之间,前面我们说Redis数据量都是存在内存中的,那如果实际业务要保存在Redis的数据量超过了单台机器的内存,这个时候最简单的方法是增加服务器内存,但是单台服务器内存不可能无限制的增加,纵向扩展不了了,便想到如何进行横向扩展.这时候我们就会想将这些业务数据分散存储在多台Redis服务器中,但是要保证多台Redis服务器能够无障碍的进行内存数据沟通,这也就是Redis集群.

#### 2、数据分区方式  
　　对于集群来说,如何将原来单台机器上的数据拆分,然后尽量均匀的分布到多台机器上,这是我们创建集群首先要考虑的一个问题,通常来说,有如下两种数据分区方式.  
　　**①、顺序分布**   
　　比如我们有100W条数据,有3台服务器,我们可以将100W/3的结果分别存储到三台服务器上。  
    **②、哈希分布**  
    同样是100W条数据,有3台服务器,通过自定义一个哈希函数,比如节点取余的方法,余数为0的存在第一台服务器,余数为1的存在第二台服务器,余数为2的存储在第三台服务器。  
    **③、两者进行对比**  
    
    | 分布方式  | 特点       | 典型应用 
    | -------| -------- | -------- 
    | 顺序分布 | 1.数据分散度高，但是容易造成访问倾斜 2.键值业务相关 3.可顺序访问 4.支持批量操作 | 1.BigTable   2.HBase   
    | 哈希分布 | 1.数据分散度高 2.键值分布业务无关 3.无法顺序访问 4.支持批量操作 | 1.Redis Cluster  2.Memcache    
 
#### 3.原生搭建   
##### ①.服务器列表
| 服务器名称  | 节点名称       | IP地址 | 端口 
| -------| -------- | -------- | ---- 
| Node1 | 主节点Node1 | 192.168.14.101   |   6379   |  
| Node2 | 主节点Node2 | 192.168.14.102   |   6380   |  
| Node3 | 主节点Node3 | 192.168.14.103   |   6381   |  
| Node4 | Node1的从节点Node4 | 192.168.14.101   |   6382   |  
| Node5 | Node2的从节点Node5 | 192.168.14.102   |   6383   |  
| Node6 | Node3的从节点Node6 | 192.168.14.103   |   6384   |  

##### ①.主要配置项
各个服务器配置文件redis.conf配置项
```$xslt
#配置端口
port ${port}
#以守护进程模式启动
daemonize yes
#pid的存放文件
pidfile /var/run/redis_${port}.pid
#日志文件名
logfile "redis_${port}.log"
#存放备份文件以及日志等文件的目录
dir "/opt/redis/data"  
#rdb备份文件名
dbfilename "dump_${port}.rdb"
#开启集群功能
cluster-enabled yes
#集群配置文件，节点自动维护
cluster-config-file nodes-${port}.conf
#集群能够运行不需要集群中所有节点都是成功的
cluster-require-full-coverage no
```

##### ②、建立各个节点通信
```aidl
redis-cli -h -p ${port1} -a ${password} cluster meet ${ip2}  ${port2} 
```

这里的 -a 参数表示该Redis节点有密码，如果没有可以不用加此参数。  
实例中的 6 个节点，分别进行如下命令：  
```aidl
redis-cli -p 6379 -a 123 cluster meet 192.168.14.101 6382
redis-cli -p 6379 -a 123 cluster meet 192.168.14.102 6380
redis-cli -p 6379 -a 123 cluster meet 192.168.14.102 6383
redis-cli -p 6379 -a 123 cluster meet 192.168.14.103 6381
redis-cli -p 6379 -a 123 cluster meet 192.168.14.103 6384
```

执行完毕后，可以查看节点通信信息：
```aidl
redis-cli -p 6379 -a 123 cluster nodes
```

#####③、分配槽位

由于我们是三主三从的架构，所以只需要对主服务器分配槽位即可。
三个节点，分配0-16383个槽位。
Node1:0~5460
Node2:5461~10922
Node3:10923~16383  
分配槽位的命令如下：  
> redis-cli -p ${port} -a ${password} cluster addslots {${startSlot}..${endSlot}}  

比如，对于Node1主节点，我们执行命令如下：
```aidl
redis-cli -p 6379 -a 123 cluster addslots {0..5462}
```  

#####④、主从配置
命令如下：  
> redis-cli -p ${port} -a {password} cluster replicate ${nodeId}  

#### 4、redis-trib构建集群  

上面原生安装Redis还是比较费劲的，在Redis安装目录的src目录下提供了redis-trib.rb脚本。
这个脚本可以用来快速搭建Redis集群。但是由于该脚本是ruby编写的，所以我们需要先安装Ruby环境。  
#####①、安装ruby
安装命令如下：
```aidl

```














