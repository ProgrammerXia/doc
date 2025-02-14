# CentOS 安装 MongoDB

## 安装流程

### 配置软件包管理系统 (`yum`)
  
创建 `/etc/yum.repos.d/mongodb-org-7.0.repo` 文件，以便直接使用 `yum` 来安装 MongoDB

```repo
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/7/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-7.0.asc
```

### 安装 MongoDB 软件包
  
```shell
yum install -y mongodb-org
```
  
禁用自动升级，将以下内容加入到 `/etc/yum.conf`

```conf
exclude=mongodb-org,mongodb-org-database,mongodb-org-server,mongodb-mongosh,mongodb-org-mongos,mongodb-org-tools
```

### 运行 MongoDB Community Edition
  
使用 `systemctl` 管理 `mongod` 服务

```shell
# 启动
systemctl start mongod
# 停止
systemctl stop mongod
# 重启
systemctl restart mongod
# 状态
systemctl stauts mongod
```

### 启用 MongoDB 用户名密码认证
  
先使用 `mongosh` 进入命令行

```shell
mongosh --host 10.225.20.21
```

进入 `admin` 数据库

```sql
use admin
```
  
创建 `admin` 用户

```javascript
db.createUser({user:"admin",pwd:"xsm@2024",roles:[{role:"root", db:"admin"}]})
```
  
配置启用认证，修改 `MongoDB` 配置文件 `/etc/mongod.conf`
  
```conf
# 开启认证
security:
  authorization: enabled
```
  
为其他数据库设置用户

```sql
-- 切换数据库, 数据库不存在自动创建
use csmar_unified_information
-- 创建只有该数据库权限的用户
db.createUser({user:"dev",pwd:"csmar@2024",roles:[{role:"readWrite",db:"csmar_unified_information"}]})
```
  
**关于开启认证后MongoDB URI 的问题** 

当启用认证后，需要注意的是 `db.auth` 这个命令，该命令必须在用户所在的数据库进行执行，例如通常 `admin` 或者 `root` 这种管理性质的用户一般都是创建在 `admin` 数据库，而针对特定数据库，可能会创建一个具有读写权限的用户，例如在 `test` 数据库，创建了一个 `test` 用户，此时如果使用 `use test`, 那么 `db.auth` 则必须使用 `test`  用户进行认证，否则提示认证失败；但是某些时候想要使用超级用户同时读写内部数据库以及普通数据库，则可以通过先在 `admin` 数据库进行 `db.auth` ，然后切换到 `test` 用户

```javascript
use test;
db.auth('test', 'test') // 授权成功, test 用户就是在 test 数据库创建的
db.auth('root', 'root')  // 提示授权失败，因为 db.auth 必须使用本数据库内的用户

use admin
db.auth('root', 'root')  // 超管用户可以在admin进行认证
use test                // 随后切换到普通数据库
db.col.countDocuments() // 执行成功
```
  
同时，如果在客户端想要连接到上述 `test` 数据库，可以使用以下2种方式：
- `mongodb://test:test@10.1.139.43:27017/test`：直接使用 `test` 用户进行连接，因为他对 `test` 数据库直接有读写权限
- `mongodb://root:root@10.1.139.43:27017/test?authSource=admin`：使用 `root` 用户连接，可以通过 `authSource` 指定授权的数据库，这样 `root` 用户可以读写 `test` 数据库了

## 将独立运行的Mogond服务转换为副本集

### 关闭服务
  
使用 `mongosh` 连接到正在运行的 `mongodb`, 然后执行关闭命令

```javascript
use admin

db.adminCommand({
    shutdown: 1,
    comment: "Convert to cluster"
 })
```

### 配置副本集
  
修改配置文件，给当前副本集设置名称

```conf
# /etc/mongod.conf
replication:
  replSetName: rs0
```

### 重启服务, 初始化副本

- 使用`systemctl start mongod` 重新启动服务
- 使用 `mongosh` 连接服务
- 初始化副本集：`rs.initiate()`

### 添加或删除节点
  
参考上面部署流程,在另外的服务器上部署新的mongodb服务
- 添加节点：`rs.add( { host: "vm3:27017" } )`
- 删除节点：`rs.remove("vm3:27017")`

##  部署分片集群
  
参考[官方部署文档](https://www.mongodb.com/docs/manual/tutorial/deploy-shard-cluster/)进行部署

###  分片集群规划
  
部署`MongoDB`分片集群需要部署以下3种服务：
- **config server**：配置服务器，用于**存储集群的元数据和配置设置**，官方建议部署至少3个节点，必须使用副本集的方式部署，即使只使用1个节点
- **shard server**：分片服务器，实际进行数据存储的服务，每个分片服务器都必须是副本集
- **mongos**：路由服务器，所有客户端发起的请求均通过 `mongos` 进行路由
  
  本次测试环境，`config server` 与 `mongos` 均部署为单节点的副本集，`shard server`部署在3台服务器，每个节点也是单节点副本集

### 部署配置服务器

> 配置服务器以及分片服务器需要的数据文件存储目录 `storage.dbPath` 需要提前创建，否则启动失败

- `config server`部署在 `/opt/module/mongodb/conf` 目录
- 创建配置服务器的配置文件 `configsvr.conf`，内容如下:

```conf
# mongodb分片集群的角色,这里是configsvr表示配置服务器
sharding:
  clusterRole: configsvr
# 配置服务器副本集名称,后续mongos配置分片时需要与这里的名称一致
replication:
  replSetName: configserver
# 设置配置服务器绑定的ip与端口
net:
  port: 27016
  bindIp: 0.0.0.0
# 配置服务器的数据文件存储目录
storage:
  dbPath: /opt/module/mongodb/conf/db
# 日志文件相关配置
systemLog:
  destination: file
  logAppend: true
  path: /opt/module/mongodb/conf/mongod.log
# 启动后在后台运行
processManagement:
  fork: true
```

- 启动配置服务器，初始化副本集信息

```shell
mongod -f configsvr.conf
```

```javascript
// 初始化
rs.initiate({
  _id: "configserver",
  configsvr: true,
  members: [
    { _id : 0, host : "10.223.16.109:27016" }
  ]
})

// 打印副本集状态
rs.status()
```

#### 部署分片副本集

- 3台服务器单副本集的 `shared server` 部署在 `/opt/module/mongodb/shard`

- 创建分片副本集的配置文件`shardsvr.conf`,内容如下(`mongodb`每种服务配置项功能一致，具体参考上方配置服务器的注释)：

```conf
sharding:
  clusterRole: shardsvr

# 这里每个shardserver都是单节点的副本集服务,所以3台服务器的replication.replSetName应该配置成不同的值
# 否则会导致3台服务器节点变成了 shardserver 3节点副本集
replication:
  replSetName: shardserver1
  # 不同服务器使用不同的值
  # replSetName: shardserver2
  # replSetName: shardserver3

net:
  bindIp: 0.0.0.0
  port: 27017

systemLog:
  destination: file
  logAppend: true
  path: /opt/module/mongodb/shard/mongod.log

storage:
  dbPath: /opt/module/mongodb/shard/db
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4

processManagement:
  fork: true
  
systemLog:
  component:
    replication:
      election:
        verbosity: -1
    accessControl:
      verbosity: -1
setParameter:
  diagnosticDataCollectionEnabled: false

```

- 在3台服务器均配置相同的信息，然后将3台服务器的分片副本集启动，最后在其中一台服务器使用 `mongosh` 进行初始化

```shell
# 3台服务器均需要启动
mongod -f shardsvr.conf
```
  
```javascript
// 在3台单副本集的分片服务器进行初始化, 不同的服务器注意对照配置文件以及服务器的域名进行配置
  
// vm2
rs.initiate({
  _id : "shardserver1",
  members: [
    { _id : 0, host : "10.223.16.108:27017" }
  ]
});

// vm3
rs.initiate({
  _id : "shardserver2",
  members: [
    { _id : 0, host : "10.223.16.109:27017" }
  ]
});

// vm4
rs.initiate({
  _id : "shardserver3",
  members: [
    { _id : 0, host : "10.223.16.110:27017" }
  ]
});
```

#### 部署 Mongos 服务

- `mongos` 部署在`/opt/module/mongodb/mongos`

- 配置文件 `/opt/module/mongodb/mongos/mongosvr.conf`

```conf
# 该配置项为 mongos 独有的配置, 格式为：配置服务器副本集名称/配置服务器ip1:port1,配置服务器ip2:port2,...
sharding:
  configDB: configserver/vm2:27016
net:
  bindIp: 0.0.0.0
  port: 27018
systemLog:
  destination: file
  logAppend: true
  path: /opt/module/mongodb/mongos/mongos.log
processManagement:
  fork: true
```

- 启动服务，并将所有的分片副本集加入到 `mongos` 服务中

```shell
# 注意，其他的服务使用`mongod`启动，只有`mongos`使用 `mongos` 命令启动
mongos -f mongosvr.conf
```

```javascript
// 将上方配置的3个单副本集的分片服务器加入到mongos中
sh.addShard("shardserver1/10.223.16.108:27017")
sh.addShard("shardserver2/10.223.16.109:27017")
sh.addShard("shardserver3/10.223.16.110:27017")
```

#### 分片集群认证

在分片集群上实施访问控制需要配置以下内容:
- 集群内部采用密钥文件进行认证
- 外部客户端使用基于角色的访问控制
  
> 这里的流程是因为先部署了未认证的 `mongodb` 分片集群，才需要停止集群后进行配置；后续部署时可以提前生成密钥文件并将配置提前写好，那这里的重启流程可以忽略，直接到最后的创建用户流程即可

##### 配置内容密钥认证

- 创建密钥文件，并设置为 `400` 权限，这里在部署集群每台服务器的`/opt/module/mongodb`位置创建：
  
  ```shell
  # 每台服务器的密钥文件必须一致,这里在其中一台生成,通过`rsync`分发到其他服务器
  openssl rand -base64 756 > keyfile
  chmod 400 keyfile
  ```

- 关闭所有的 `mongos` 、`config server`、`shard server` 服务
  
```shell
#27018 是 mongos 服务器的端口
mongosh --port 27018
```

```javascript
// 先停止负载均衡器
sh.stopBalancer()
// 检查负载均衡器是否停止,未停止前不要进行其他操作
sh.getBalancerState()
// 关闭 mongos 服务器
db.getSiblingDB("admin").shutdownServer()
```

```shell
#27016是 config server的端口
mongosh --port 27016
```
  
```javascript
// 如果配置服务器是多台,则先关闭secondary服务，最后关闭primary服务
db.getSiblingDB("admin").shutdownServer()
```

```shell
# 27017 是 shard server 端口
mongosh --port 27017
```

```javascript
db.getSiblingDB("admin").shutdownServer()
```

- 在所有服务的配置文件中新增密钥文件配置，修改 `mongs` 配置文件 `mongosvr.conf`, `config server` 配置文件 `configsvr.conf` 以及所有分片副本集的配置文件 `shardsvr.conf` :
  
```conf
# 以上列举的所有文件添加密钥文件配置项
security:
  keyFile: /opt/module/mongodb/keyfile
```
  
配置完成后，依次启动 `config server` ， `shard server` , `mongos`
  
```shell
mongod -f configsvr.conf
mongod -f shardsvr.conf
mongos -f mongosvr.conf
```

##### 配置认证用户

- 连接 `mongos` 服务，启动负载均衡器，配置用户
  
```shell
mongosh --port 27018
```
  
```javascript
// 重新启动负载均衡器
sh.startBalancer()
// 检查是否返回true
sh.getBalancerState()
  
// 创建具有管理员权限的用户，这里创建具有所有权限 root 角色的 root 用户; 注意用户需要在admin数据库创建
use admin
db.createUser({user:"root",pwd:"mongo#!2024",roles:[{role:"root",db:"admin"}]})
  
// 此时，大部分操作就需要进行认证后才能继续进行
db.auth('root', 'mongo#!2024')
  
// 创建一个普通用户，只对 csmar_unified_information 数据库拥有读写权限; 注意如果是只想要该用户对指定数据库的权限，则先需要切换到指定数据库
use csmar_unified_information;
db.createUser({user:"dev",pwd:"csmar@2024",roles:[{role:"readWrite",db:"csmar_unified_information"}]})
  
db.auth('dev', 'csmar@2024')
db.news_source.countDocuments()
```

- 通常在分片集群，大部分操作都通过 `mongos` 服务进行，如果需要登录到分片服务器的某一台服务进行操作，那么分片服务器也需要进行认证用户：
  
```shell
mongosh --port 27017
```
  
`mongos` 中实际创建的用户保存在 `config server` 中，也就是在 `mongos` 、`config server` 可以使用同一份用户数据；但是这些用户在 `shard server` 不可用，必须手动创建；为了减少歧义，也可以使用相同的用户名进行用户创建授权

```JavaScript
use admin
db.createUser({user:"root",pwd:"root",roles:[{role:"root",db:"admin"}]})
use online;
db.createUser({user:"dev",pwd:"dev",roles:[role:"readWrite",db:"online"]})
```

#### Collection 分片
  
如果一个 collection 没有数据，则直接使用以下命令进行分片；如果已经包含数据了，则先需要对进行分片的字段创建索引；

```javascript
// 格式如下
sh.shardCollection("<database>.<collection>", { <shard key field> : 1, ... } )

sh.shardCollection("csmar_unified_information.news_source", { 'title':1,'url':1,'release_time':1 })
```
  
对已经分片的集合,添加新的字段到分片键

```javascript
db.adminCommand({
 refineCollectionShardKey: "online.news_source",
 key: { url: 1, title: 1, create_time: 1 }
})
```
  
对已经分片的集合,进行重新分片

```javascript
db.adminCommand({
reshardCollection: "online.news_source",
key: {create_time:1, rules_update_time:1}
})
```
  
重新分片需要一定时间，使用以下查询进行实时查看进度
  
```javascript
db.getSiblingDB("admin").aggregate([
{ $currentOp: { allUsers: true, localOps: false } },
{
  $match: {
    type: "op",
    "originatingCommand.reshardCollection": "online.news_source"
  }
}
])
```
  
  上述查询结果字段简析：
- `totalOperationTimeElapsedSecs`：经过的操作时间（以秒为单位）
- `remainingOperationTimeEstimatedSecs`：当前重新分片操作的估计剩余时间（以秒为单位）。当新的重新分片操作开始时，它会以 `-1` 返回。
  
重新分片操作按顺序执行以下阶段：

- 克隆阶段复制当前的集合数据。
- 追赶阶段将所有待处理的写入操作应用于重新分片的集合。

`remainingOperationTimeEstimatedSecs`：设置为悲观的时间估计值：
- 将追赶阶段时间估计值设置为克隆阶段时间，这是一个相对较长的时间。
- 实际上，如果只有几个待处理的写入操作，则实际的追赶阶段时间相对较短
  
重新分片进程完成后，重新分片命令会返回 `ok: 1`
	  
```javascript
{
ok: 1,
'$clusterTime': {
clusterTime: Timestamp({ t: 1721876797, i: 125 }),
signature: {
hash: Binary.createFromBase64('A/eSsT16LDiYxn+TCBMx1V2ZAsM=', 0),
keyId: Long('7395132144030318615')
}
},
operationTime: Timestamp({ t: 1721876797, i: 125 })
}
```

## MongoDB 数据迁移
  
  使用 `mongodump` 导出指定的 `collection`
  
```shell
mongodump --uri mongodb://dev:csmar%402024@10.223.16.110:27018/csmar_unified_information -c news_source -o output -q='{"create_time":{"$lte":{"$date":"2024-11-29T23:59:59.999Z"},"$gte":{"$date":"2024-11-29T12:36:30.908Z"}}}'
```
  
- `--uri`: 配置连接字符串
- `-c`: 配置要导出的集合
- `-o`：配置导出的目的地(文件夹, 生成的Bson文件会以该集合所在数据库为次级文件夹,  上述命令执行的最终结果就是 `output/csmar_unified_information/new_source.bson` )
- `-q`：配置额外的查询条件, 这里是使用`_id` 进行查询, json 格式必须采用[官方指定的格式](https://www.mongodb.com/zh-cn/docs/manual/reference/mongodb-extended-json/#mongodb-bsontype-ObjectId)，这里`_id` 为 `ObjectId` 类型, 使用 `"$oid":"objectid字符串"` 格式
- 该命令不是必须在 `10.225.20.21` 这台服务器执行，只要某台服务器安装了 `mongodb` 并且可以访问该服务器就可以执行该命令并导出bson文件
  
使用 `mongorestore` 还原 `mongodump` 导出的数据：
- 使用 `--uri` 配置连接信息，`--uri` 可以省略，配置mongodb的连接信息，这里因为是导入分片集群故连接地址填写的是mongos 服务的 ip:port, 如果是单机版，直接使用 mongodb 服务的 ip:port 即可
- `csmar_unified_information` 是执行命令目录下的文件夹(数据库)，存放 `mongodump` 产生的 `bson` 文件目录
  
```shell
mongorestore mongodb://dev:csmar%402024@10.223.16.110:27018/csmar_unified_information output/csmar_unified_information

mongorestore mongodb://dev:dev@10.1.139.43:27018/online output/csmar_unified_information
```
