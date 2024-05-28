---
title: Redis集群部署指南
urlname: nsgd1vcw40dhn1gz
date: '2024-03-28 16:31:22'
updated: '2024-03-28 16:31:40'
description: "本章是基于CentOS7下的Redis集群教程，包括：单机安装RedisRedis主从Redis分片集群1.单机安装Redis首先需要安装Redis所需要的依赖：yum install -y gcc tcl然后将课前资料提供的Redis安装包上传到虚拟机的任意目录：\U0001F4CEredis-6.2.4...."
---
本章是基于CentOS7下的Redis集群教程，包括：

- 单机安装Redis
- Redis主从
- Redis分片集群
# 1.单机安装Redis
首先需要安装Redis所需要的依赖：
```shell
yum install -y gcc tcl
```
然后将课前资料提供的Redis安装包上传到虚拟机的任意目录：[redis-6.2.4.tar.gz](https://www.yuque.com/attachments/yuque/0/2024/gz/29688613/1711614690538-8492e392-5d23-47a1-9e1d-2f0a4f564254.gz?_lake_card=%7B%22src%22%3A%22https%3A%2F%2Fwww.yuque.com%2Fattachments%2Fyuque%2F0%2F2024%2Fgz%2F29688613%2F1711614690538-8492e392-5d23-47a1-9e1d-2f0a4f564254.gz%22%2C%22name%22%3A%22redis-6.2.4.tar.gz%22%2C%22size%22%3A2457940%2C%22ext%22%3A%22gz%22%2C%22source%22%3A%22%22%2C%22status%22%3A%22done%22%2C%22download%22%3Atrue%2C%22taskId%22%3A%22u44084d11-f344-4b50-9d6e-8d3c6146640%22%2C%22taskType%22%3A%22transfer%22%2C%22type%22%3A%22application%2Fx-gzip%22%2C%22mode%22%3A%22title%22%2C%22id%22%3A%22uce192a13%22%2C%22card%22%3A%22file%22%7D)
例如，我放到了/tmp目录：
![image-20210629114830642.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/61e2e3cf3c5f34e390148a412e17c825.png)
解压缩：
```shell
tar -xvf redis-6.2.4.tar.gz
```
解压后：
![image-20210629114941810.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/db679d8fab0cf9142929598d44665be9.png)
进入redis目录：
```shell
cd redis-6.2.4
```
运行编译命令：
```shell
make && make install
```
如果没有出错就安装成功。然后修改redis.conf文件中的一些配置（**注意是四个0，不要额外的**）：
```properties
# 绑定地址，默认是127.0.0.1，会导致只能在本地访问。修改为0.0.0.0则可以在任意IP访问
bind 0.0.0.0
# 数据库数量，设置为1
databases 1
```
启动Redis：
```shell
redis-server redis.conf
```
停止redis服务（ctrl + c也可停止）：
```shell
redis-cli shutdown
```
# 2.Redis主从集群
## 2.1.集群结构
我们搭建的主从集群结构如图：
![image-20210630111505799.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/8f65e4b0c5a5b82e11b84f4c6d39586d.png)
共包含三个节点，一个主两个从。这里我们会在同一台虚拟机中开启3个redis实例，模拟主从集群，信息如下：

| IP | PORT | 角色 |
| --- | --- | --- |
| 192.168.206.129 | 7001 | master |
| 192.168.206.129 | 7002 | slave |
| 192.168.206.129 | 7003 | slave |

## 2.2.准备实例和配置
要在同一台虚拟机开启3个实例，必须准备三份不同的配置文件和目录，配置文件所在目录也就是工作目录。
确保下面的配置已经更改（位置也别错）
![image.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/20291adb5413ac3f3ce8723738c3032d.png)
### 1）创建目录
我们创建三个文件夹，名字分别叫7001、7002、7003：
```shell
# 进入/tmp目录
cd /tmp
# 创建目录
mkdir 7001 7002 7003
```
如图：
![image-20210630113929868.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/7d39b97eab2330b1bfdd7c6383f033f3.png)
### 2）恢复原始配置
修改redis-6.2.4/redis.conf文件，将其中的持久化模式改为默认的RDB模式，AOF保持关闭状态。
```properties
# 开启RDB
# save ""
save 3600 1
save 300 100
save 60 10000

# 关闭AOF
appendonly no
```
### 3）拷贝配置文件到每个实例目录
然后将redis-6.2.4/redis.conf文件拷贝到三个目录中（在/tmp目录执行下列命令）：
```shell
# 方式一：逐个拷贝
cp redis-6.2.4/redis.conf 7001
cp redis-6.2.4/redis.conf 7002
cp redis-6.2.4/redis.conf 7003
# 方式二：管道组合命令，一键拷贝
echo 7001 7002 7003 | xargs -t -n 1 cp redis-6.2.4/redis.conf
```
### 4）修改每个实例的端口、工作目录
修改每个文件夹内的配置文件，将端口分别修改为7001、7002、7003，将rdb文件保存位置都修改为自己所在目录（在/tmp目录执行下列命令）：
```shell
sed -i -e 's/6379/7001/g' -e 's/dir .\//dir \/tmp\/7001\//g' 7001/redis.conf
sed -i -e 's/6379/7002/g' -e 's/dir .\//dir \/tmp\/7002\//g' 7002/redis.conf
sed -i -e 's/6379/7003/g' -e 's/dir .\//dir \/tmp\/7003\//g' 7003/redis.conf
```
### 5）修改每个实例的声明IP（注意IP需要修改）
虚拟机本身有多个IP，为了避免将来混乱，我们需要在redis.conf文件中指定每一个实例的绑定ip信息，格式如下(**下面这个只是语法说明，别执行**)：
```properties
# redis实例的声明 IP
replica-announce-ip 192.168.150.101
```
每个目录都要改，我们一键完成修改（在/tmp目录执行下列命令）：
```shell
# 逐一执行
sed -i '1a replica-announce-ip 192.168.206.129' 7001/redis.conf
sed -i '1a replica-announce-ip 192.168.206.129' 7002/redis.conf
sed -i '1a replica-announce-ip 192.168.206.129' 7003/redis.conf

# 或者一键修改
printf '%s\n' 7001 7002 7003 | xargs -I{} -t sed -i '1a replica-announce-ip 192.168.206.130' {}/redis.conf
```
## 2.3.启动
为了方便查看日志，我们打开3个ssh窗口，分别启动3个redis实例，启动命令（在/tmp目录执行下列命令）：
```shell
# 第1个
redis-server 7001/redis.conf
# 第2个
redis-server 7002/redis.conf
# 第3个
redis-server 7003/redis.conf
```
启动后：
![image-20210630183914491.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/a9ad78596d6d44af892693ccf689461d.png)
如果要一键停止，可以运行下面命令：
```shell
printf '%s\n' 7001 7002 7003 | xargs -I{} -t redis-cli -p {} shutdown
```
## 2.4.开启主从关系
现在三个实例还没有任何关系，要配置主从可以使用replicaof 或者slaveof（5.0以前）命令。
**新开窗口执行下述命令，一旦ctrl +c退出后，redis服务将中断**
有临时和永久两种模式：

-  修改配置文件（永久生效） 
   - 在redis.conf中添加一行配置：`slaveof <masterip> <masterport>`
-  使用redis-cli客户端连接到redis服务，执行slaveof命令（重启后失效）： 
```shell
slaveof <masterip> <masterport>
```
**注意**：在5.0以后新增命令replicaof，与salveof效果一致。（**注意IP需要修改**）
这里我们为了演示方便，使用方式二（在/tmp目录执行下列命令）。通过redis-cli命令连接7002，执行下面命令：
```shell
# 连接 7002
redis-cli -p 7002
# 执行slaveof
slaveof 192.168.206.129 7001
```
执行完：exit，退出可继续执行后续命令。通过redis-cli命令连接7003，执行下面命令：
![image.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/9e238baf1a7bc7e077540256b4347553.png)
```shell
# 连接 7003
redis-cli -p 7003
# 执行slaveof
slaveof 192.168.206.130 7001
```
然后连接 7001节点，查看集群状态：
```shell
# 连接 7001
redis-cli -p 7001
# 查看状态
info replication
```
结果：
![image-20210630201258802.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/57b9dd6d8cf9b9f9fba9287436073f28.png)
## 2.5.测试
执行下列操作以测试：

-  利用redis-cli连接7001，执行`set num 123` 
-  利用redis-cli连接7002，执行`get num`，再执行`set num 666` 
-  利用redis-cli连接7003，执行`get num`，再执行`set num 888` 

可以发现，只有在7001这个master节点上可以执行写操作，7002和7003这两个slave节点只能执行读操作。
# 3.搭建哨兵集群
## 3.1.集群结构
这里我们搭建一个三节点形成的Sentinel集群，来监管之前的Redis主从集群。如图：
![image-20210701215227018.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/1e5af1a3574b01135b8c9b869893524b.png)
三个sentinel实例信息如下：

| 节点 | IP | PORT |
| --- | --- | --- |
| s1 | 192.168.206.129 | 27001 |
| s2 | 192.168.206.129 | 27002 |
| s3 | 192.168.206.129 | 27003 |

## 3.2.准备实例和配置
要在同一台虚拟机开启3个实例，必须准备三份不同的配置文件和目录，配置文件所在目录也就是工作目录。
我们创建三个文件夹，名字分别叫s1、s2、s3：
```shell
# 进入/tmp目录
cd /tmp
# 创建目录
mkdir s1 s2 s3
```
如图：
![image-20210701215534714.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/c77f78f4ff9b4d68c46bf371af4bdf3b.png)
然后我们在s1目录创建一个sentinel.conf文件，添加下面的内容：

- cd /tmp/s1
- touch sentinel.conf
- vi sentinel.conf
- 粘贴下述文本
```properties
port 27001
sentinel announce-ip 192.168.206.130
sentinel monitor mymaster 192.168.206.130 7001 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
dir "/tmp/s1"
```
解读：

- `port 27001`：是当前sentinel实例的端口
- `sentinel monitor mymaster 192.168.206.129 7001 2`：指定主节点信息 
   - `mymaster`：主节点名称，自定义，任意写
   - `192.168.206.129 7001`：主节点的ip和端口
   - `2`：选举master时的quorum值

然后将s1/sentinel.conf文件拷贝到s2、s3两个目录中（在**/tmp**目录执行下列命令）：
```shell
# 方式一：逐个拷贝
cp s1/sentinel.conf s2
cp s1/sentinel.conf s3
# 方式二：管道组合命令，一键拷贝
echo s2 s3 | xargs -t -n 1 cp s1/sentinel.conf
```
修改s2、s3两个文件夹内的配置文件，将端口分别修改为27002、27003：
```shell
sed -i -e 's/27001/27002/g' -e 's/s1/s2/g' s2/sentinel.conf
sed -i -e 's/27001/27003/g' -e 's/s1/s3/g' s3/sentinel.conf
```
## 3.3.启动
为了方便查看日志，我们打开3个ssh窗口，分别启动3个redis实例，启动命令（在tmp目录执行）：
```shell
# 第1个
redis-sentinel s1/sentinel.conf
# 第2个
redis-sentinel s2/sentinel.conf
# 第3个
redis-sentinel s3/sentinel.conf
```
启动后：![image-20210701220714104.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/da975ead6f6cca8eaed8d728921047f3.png)
## 3.4.测试

- 停止7001（当前主节点），此时会在多个哨兵中选举出一个哨兵的master，如下
   - 27001投给27002
   - 27002投给自己
   - 27003投给27002
   - 因此27002变成哨兵master

![步骤1-主观下线-选举哨兵master.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/7a5b48eb62b51aa5f3c98b64390e3462.png)

- 主观下线变客观下线

![步骤2-主观下线变客观下线.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/2af50e6612746013e34956534a18c0ea.png)

- 哨兵master自己从当前集群中的slave节点中选出一个新的master，原则是：
```properties
1-首先会判断slave节点与master节点断开时间长短，如果超过指定值（down-after-milliseconds * 10）则会排除该slave节点
2-然后判断slave节点的slave-priority值，越小优先级越高，如果是0则永不参与选举
3-如果slave-prority一样，则判断slave节点的offset值，越大说明数据越新，优先级越高
4-最后是判断slave节点的运行id大小，越小优先级越高。
```
![步骤3-哨兵master选举新的主节点-新主节点执行slaveof no one.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/8d75d2ab1c7e3bbb308b755af67a8584.png)

- 现有slave节点执行slave of，服从新的master节点

![步骤4-现有slave节点重新分配master节点.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/5c36248dff3418e5c1e2b129a5bf098d.png)

- 其余哨兵得到哨兵msater的选举通知

![步骤5-其余哨兵得到通知.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/7f5b4c475c81283b916f8a581132d1da.png)

- 此时我们恢复7001，会发现会做一次全量同步

![步骤6-7001恢复启动并实现数据全量同步.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/b45d123727404b4369712282ab0e01f5.png)

- 新加入的节点，执行slave of服从当前master（此时是从配置文件中读取）

![步骤7-哨兵监控执行从节点命令.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/8cb99a34b17cde4a44ca8186a584b965.png)
# 4.搭建分片集群
## 4.1.集群结构
分片集群需要的节点数量较多，这里我们搭建一个最小的分片集群，包含3个master节点，每个master包含一个slave节点，结构如下
![image-20210702164116027.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/2d8653c6e68d916e7e283872cdd35910.png)
这里我们会在同一台虚拟机中开启6个redis实例，模拟分片集群，信息如下：

| IP | PORT | 角色 |
| --- | --- | --- |
| 192.168.206.129 | 7001 | master |
| 192.168.206.129 | 7002 | master |
| 192.168.206.129 | 7003 | master |
| 192.168.206.129 | 8001 | slave |
| 192.168.206.129 | 8002 | slave |
| 192.168.206.129 | 8003 | slave |

## 4.2.准备实例和配置
停止原来的所有redis服务（哨兵也要停止），同时删除之前的7001、7002、7003这几个目录，重新创建出7001、7002、7003、8001、8002、8003目录：
```shell
# 进入/tmp目录
cd /tmp
# 删除旧的，避免配置干扰
rm -rf 7001 7002 7003
# 创建目录
mkdir 7001 7002 7003 8001 8002 8003
```
在**/tmp**下准备一个新的redis.conf文件，内容如下：

- touch redis.conf
- vi redis.conf
```properties
port 6379
# 开启集群功能
cluster-enabled yes
# 集群的配置文件名称，不需要我们创建，由redis自己维护
cluster-config-file /tmp/6379/nodes.conf
# 节点心跳失败的超时时间
cluster-node-timeout 5000
# 持久化文件存放目录
dir /tmp/6379
# 绑定地址(非常重要，别漏也别改)
bind 0.0.0.0
# 让redis后台运行
daemonize yes
# 注册的实例ip
replica-announce-ip 192.168.206.130
# 保护模式
protected-mode no
# 数据库数量
databases 1
# 日志
logfile /tmp/6379/run.log
```
![image.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/4ad07816f9fee60c006a0a31dbc0554c.png)
将这个文件拷贝到每个目录下：
```shell
# 进入/tmp目录
cd /tmp
# 执行拷贝
echo 7001 7002 7003 8001 8002 8003 | xargs -t -n 1 cp redis.conf
```
修改每个目录下的redis.conf，将其中的6379修改为与所在目录一致：
```shell
# 进入/tmp目录
cd /tmp
# 修改配置文件
printf '%s\n' 7001 7002 7003 8001 8002 8003 | xargs -I{} -t sed -i 's/6379/{}/g' {}/redis.conf
```
此时可以进到任意目录查看一下，确认是否都存在相关的配置文件
## 4.3.启动
因为已经配置了后台启动模式，所以可以直接启动服务：
```shell
# 进入/tmp目录
cd /tmp
# 一键启动所有服务
printf '%s\n' 7001 7002 7003 8001 8002 8003 | xargs -I{} -t redis-server {}/redis.conf
```
通过ps查看状态：
```shell
ps -ef | grep redis
```
发现服务都已经正常启动：![image-20210702174255799.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/2d6a1af50692c14747e6309d0041e377.png)
如果要关闭所有进程，可以执行命令：
```shell
ps -ef | grep redis | awk '{print $2}' | xargs kill
```
或者（推荐这种方式）：
```shell
printf '%s\n' 7001 7002 7003 8001 8002 8003 | xargs -I{} -t redis-cli -p {} shutdown
```
## 4.4.创建集群

---

- 更多测试细节也可查看：[https://www.yuque.com/xiankanpengyouquandisitiaodongtai/diods0/fzoohtlg0r8gz6cd#ecac7084](https://www.yuque.com/xiankanpengyouquandisitiaodongtai/diods0/fzoohtlg0r8gz6cd#ecac7084)

---

虽然服务启动了，但是目前每个服务之间都是独立的，没有任何关联。
我们需要执行命令来创建集群，在Redis5.0之前创建集群比较麻烦，5.0之后集群管理命令都集成到了redis-cli中。
1）Redis5.0之前（**我们版本是之后，不适用这个**）
Redis5.0之前集群命令都是用redis安装包下的src/redis-trib.rb来实现的。因为redis-trib.rb是有ruby语言编写的所以需要安装ruby环境。
```shell
# 安装依赖
yum -y install zlib ruby rubygems
gem install redis
```
然后通过命令来管理集群：
```shell
# 进入redis的src目录
cd /tmp/redis-6.2.4/src
# 创建集群
./redis-trib.rb create --replicas 1 192.168.150.101:7001 192.168.150.101:7002 192.168.150.101:7003 192.168.150.101:8001 192.168.150.101:8002 192.168.150.101:8003
```
2）Redis5.0以后
我们使用的是Redis6.2.4版本，集群管理以及集成到了redis-cli中，格式如下：
```shell
redis-cli --cluster create --cluster-replicas 1 192.168.206.129:7001 192.168.206.129:7002 192.168.206.129:7003 192.168.206.129:8001 192.168.206.129:8002 192.168.206.129:8003
```
命令说明：

- `redis-cli --cluster`或者`./redis-trib.rb`：代表集群操作命令
- `create`：代表是创建集群
- `--replicas 1`或者`--cluster-replicas 1` ：指定集群中每个master的副本个数为1，此时`节点总数 ÷ (replicas + 1)` 得到的就是master的数量。因此节点列表中的前n个就是master，其它节点都是slave节点，随机分配到不同master

运行后的样子：
![image-20210702181101969.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/dda545f4f4aaf1acf358631d4d822acd.png)
这里输入yes，则集群开始创建：
![image-20210702181215705.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/160d6ee5accdfe853860588cafc917cc.png)
![创建分片集群时确认集群关系图.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/bdcd54294c0a6e37cf4c6749cf8c98f5.png)通过命令可以查看集群状态（下面端口可以是集群中的任意端口都行）：
```shell
redis-cli -p 7001 cluster nodes
```
![image-20210702181922809.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/5568a6e6f7d4680b6285b5a026fd243c.png)
其中的映射关系分析如下图
![分片集群日志分析.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/40d9f58f63f26cbb4ab40eb80e20ecab.png)

## 4.5.测试
尝试连接7001节点，存储一个数据：
```shell
# 连接
redis-cli -p 7001
# 存储数据
set num 123
# 读取数据
get num
# 再次存储
set a 1
```
结果悲剧了：
![](assets/image-20210702182343979.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_9%2Ctext_5rK554K45bCP5rOi%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10#id=FH3Bq&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
集群操作时，需要给`redis-cli`**加上**`**-c**`**参数**才可以：
```shell
 redis-cli -c -p 7001
```
这次可以了：
![image-20210702182602145.png](https://raw.githubusercontent.com/choodsire666/blog-img/main/Redis集群部署指南/30287c0894031a88e1e14aa2d323c7da.png)
