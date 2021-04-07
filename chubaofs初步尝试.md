### chubaofs 初步尝试



###  硬件

|  节点   | 内存  |CPU|磁盘|
|  ----  | ----  |----  |----  |
| 172.16.84.37  | 125G |72逻辑核 |2*7T HDD，1*3.5T SSD |
| 172.16.84.41  | 125G |72逻辑核 |2*7T HDD，1*3.5T SSD |
| 172.16.84.42  | 125G |72逻辑核 |2*7T HDD，1*3.5T SSD |

### chubaofs编译

```
$ git clone http://github.com/chubaofs/chubaofs.git
$ cd chubaofs
$ make build
```
### 版本

- OS
```
[root@node1 ~]# uname -r
3.10.0-1160.15.2.el7.x86_64
```
- chubaofs

```
[root@node1 ~]# cfs-cli --version
ChubaoFS CLI
Version : v2.3.0
Branch  : HEAD
Commit  : 3c807718db03ef397d1a78fd0f1a41a48c589f2e
Build   : go1.15.5 linux amd64 2021-04-06 18:21
```

### master节点
- 核心功能
	- Master负责管理ChubaoFS整个集群，主要存储5种元数据，包括：数据节点、元数据节点、卷、数据分片、元数据分片。所有的元数据都保存在master的内存中，并且持久化到RocksDB。 多个Master之间通过raft协议保证集群元数据的一致性。注意：master的实例最少需要3个
	- 多租户，资源隔离;多个卷共享数据节点和元数据节点,每个卷独享各自的数据分片和元数据分片与数据节点和元数据节点即有同步交互，也有异步交互，交互方式与任务类型相关。
- 配置
```
// master.json，每个节点部署一个master,基于raft协议，配置文件中node1的ID为1，node2的ID是2，node3的id是3
{
  "role": "master",
  "ip": "172.16.84.37",
  "listen": "17010",
  "prof":"17020",
  "id":"1",
  "peers": "1:172.16.84.37:17010,2:172.16.84.41:17010,3:172.16.84.42:17010",
  "retainLogs":"20000",
  "logDir": "/cfs/master/log",
  "logLevel":"info",
  "walDir":"/cfs/master/data/wal",
  "storeDir":"/cfs/master/data/store",
  "consulAddr": "http://172.16.84.37",
  "exporterPort": 9500,
  "clusterName":"chubaofs01",
  "metaNodeReservedMem": "1073741824"
}
```
- 启动

```
// 172.16.84.37、 172.16.84.41、 172.16.84.42部署一个,每个节点的master.json配置的ID不同
nohup  /usr/bin/cfs-server -f -c /root/chubaofs-cfg/master.json      &
```

### metadata节点
- 功能
	- 元数据节点是由多个元数据分片(meta partition)和基于multiRaft的对应个数的raft实例组成； 每个元数据分片(meta partition)都是一个inode范围，且包含两个内存BTrees： inode BTree 和dentry BTree。注意：MetaNode的实例最少需要3个
- 配置

```
[root@node1 chubaofs-cfg]# cat metanode.json 
{
    "role": "metanode",
    "listen": "17210",
    "prof": "17220",
    "logLevel": "info",
    "metadataDir": "/cfs/metanode/data/meta",
    "logDir": "/cfs/metanode/log",
    "raftDir": "/cfs/metanode/data/raft",
    "raftHeartbeatPort": "17230",
    "raftReplicaPort": "17240",
    "totalMem":  "8589934592",
    "consulAddr": "http://172.16.84.37",
    "exporterPort": 9501,
    "masterAddr": [
        "172.16.84.37:17010",
        "172.16.84.41:17010",
        "172.16.84.42:17010"
    ]
}
```

- 启动
```
// 172.16.84.37、 172.16.84.41、 172.16.84.42部署一个,每个节点的master.json配置的ID不同
nohup  /usr/bin/cfs-server -f -c /root/chubaofs-cfg/metanode.json      &
```
### objectnode节点
- 功能
	- 通过执行ChubaoFS的二进制文件并用“-c”参数指定的配置文件来启动一个ObjectNode进程。如果不打算使用对象存储功能，无需启动ObjectNode节点。


- 配置

```
[root@node1 chubaofs-cfg]# cat objectnode.json 
{
    "role": "objectnode",
    "domains": [
        "node1"
    ],
    "listen": 17410,
    "masterAddr": [
        "172.16.84.37:17010",
        "172.16.84.41:17010",
        "172.16.84.42:17010"
    ],
    "logLevel": "info",
    "logDir": "/cfs/Logs/objectnode"
}
```
- 启动
```
// 172.16.84.37、 172.16.84.41、 172.16.84.42部署一个,每个节点的metanode.json配置的domains为本机的主机名
nohup  /usr/bin/cfs-server -f -c /root/chubaofs-cfg/metanode.json      &
```

### datanode节点
- 功能
	- 通过执行ChubaoFS的二进制文件并用“-c”参数指定的配置文件来启动一个DATANODE进程。注意datanode的实例最少需要４个。
- 配置
```
[root@node1 chubaofs-cfg]# cat datanode.json 
{
  "role": "datanode",
  "listen": "17310",
  "prof": "17320",
  "logDir": "/cfs/datanode/log",
  "logLevel": "info",
  "raftHeartbeat": "17330",
  "raftReplica": "17340",
  "raftDir":"/cfs/datanode/log",
  "consulAddr": "http://172.16.84.37",
  "exporterPort": 9502,
  "masterAddr": [
        "172.16.84.37:17010",
        "172.16.84.41:17010",
        "172.16.84.42:17010"
  ],
  "disks": [
     "/cfs/data0:10737418240",
     "/cfs/data1:10737418240"
 ]
}
```
- 启动
```
// 172.16.84.37、 172.16.84.41、 172.16.84.42部署一个
nohup  /usr/bin/cfs-server -f -c /root/chubaofs-cfg/datanode.json      &
```

### 客户端挂载
- 配置集群信息
```
//配置 客户端挂载的配置client.json
{
  "mountPoint": "/cfs/mountpoint",
  "volName": "vol1",
  "owner": "tester",
  "masterAddr": "172.16.84.37:17010,172.16.84.41:17010,172.16.84.42:17010",
  "logDir": "/cfs/client/log",
  "profPort": "17510",
  "exporterPort": "9504",
  "logLevel": "info"
}

```

- 挂载客户端
```
nohup cfs-client -c client.json  & 
```
### 集群操作
- 配置集群信息
```
//编辑当前主机用户下的.cfs-cli.json

{
  "masterAddr": [
         "172.16.84.37:17010",
        "172.16.84.41:17010",
        "172.16.84.42:17010"
  ],
  "timeout": 10
}
```
- 查看集群状态
```
[root@node1 ~]# cfs-cli cluster info
[Cluster]
  Cluster name       : chubaofs01
  Master leader      : 172.16.84.37:17010
  Auto allocate      : Enabled
  MetaNode count     : 3
  MetaNode used      : 32 GB
  MetaNode total     : 24 GB
  DataNode count     : 3
  DataNode used      : 2260 GB
  DataNode total     : 44640 GB
  Volume count       : 2
  BatchCount         : 0
  MarkDeleteRate     : 0
  DeleteWorkerSleepMs: 0
  AutoRepairRate     : 0
```
- 创建卷
```
// 创建900G的卷
cfs-cli volume create  vol3 tester --capacity 900
```
- 查看卷信息
```
[root@node1 ~]# cfs-cli volume list
VOLUME                                                             OWNER                   USED        TOTAL       STATUS      CREATE TIME
vol1                                                               tester                  240 GB      240 GB      Normal      Wed, 07 Apr 2021 12:59:30 CST
vol3                                                               tester                  143 GB      900 GB      Normal      Wed, 07 Apr 2021 16:35:05 CST
[root@node1 ~]# cfs-cli volume info vol3
Summary:
  ID                   : 11
  Name                 : vol3
  Owner                : tester
  Zone                 : default
  Status               : Normal
  Capacity             : 900 GB
  Create time          : 2021-04-07 16:35:05
  Authenticate         : Disabled
  Follower read        : Enabled
  Enable token         : Disabled
  Cross zone           : Disabled
  Inode count          : 12350741
  Dentry count         : 12350738
  Max metaPartition ID : 12
  Meta partition count : 3
  Meta replicas        : 3
  Data partition count : 10
  Data replicas        : 3
```
