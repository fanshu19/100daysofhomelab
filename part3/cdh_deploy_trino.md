# CDH集成Trino (presto)

# Trino是什么？

Trino的前身是PrestoSQL, 它是一个开源的分布式SQL查询引擎，它能够高效查询不同种类和各种规模的数据源，还实现了跨数据源查询。

提到PrestoSQL，就会有PrestoDB，它们都是从Presto演变而来，其中的分家原因大家感兴趣可以搜索了解一下, 文章底部我也提供了一些资料供参考。

# Trino集群安装

## 节点规划

|  hostname   | 	节点用途  |
|  :----  | :----  |
| bigdata-node1  | coordinator |
| bigdata-node2  | Worker |
| bigdata-node3  | Worker |

trino版本：400

由于我安装了最新版，安装步骤还是要看一下[官网文档](https://trino.io/docs/current/installation/deployment.html)的。

## 软件包安装

### 安装jdk17 (全部节点)

由于集群环境中已经存在低版本jdk1.8，为了避免版本更新对已有项目的影响，所以在各台服务器新增用户`presto，在新的用户中安装jdk17，并且在该新用户的环境变量配置jdk17。

- 新建用户`presto`，密码也是`presto`
```
useradd presto
passwd presto
```

trino官方推荐的jdk17版本是Azul Zulu，而且trino 400版本，对JDK版本也有要求，最低要求17.0.3，早期版本JDK8或者JDK11不会起效，最新版本JDK18或者JDK19不支持，可能有效，但是官方没有通过测试
```
# 找一个文件夹存放安装包
wget https://cdn.azul.com/zulu/bin/zulu17.36.13-ca-jdk17.0.4-linux_x64.tar.gz

tar -zxvf zulu17.36.13-ca-jdk17.0.4-linux_x64.tar.gz -C /usr/java/
cd /usr/java
mv zulu17.36.13-ca-jdk17.0.4-linux_x64 jdk17.0.4
chown -R presto:presto jdk17.0.4
```

修改`presto`用户的配置文件，增加jdk11的环境变量
vim /home/presto/.bash_profile
```
export JAVA_HOME=/usr/java/jdk17.0.4
export JRE_HOME=$JAVA_HOME/jre
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib/rt.jar
```

切换到`presto`用户，查看jdk版本
```
su presto
source ~/.bash_profile
java -version

[presto@bigdata-node1 java]$ java -version
openjdk version "17.0.4" 2022-07-19 LTS
OpenJDK Runtime Environment Zulu17.36+13-CA (build 17.0.4+8-LTS)
OpenJDK 64-Bit Server VM Zulu17.36+13-CA (build 17.0.4+8-LTS, mixed mode, sharing)
```

### 安装trino (coordinator节点)

下载地址：wget https://repo1.maven.org/maven2/io/trino/trino-server/400/trino-server-400.tar.gz
```
tar -xzvf trino-server-400.tar.gz -C /opt/
chown -R presto:presto /opt/trino-server-400
```

创建指定目录
```
# 切换presto用户
su presto

# 创建日志存储目录
mkdir /opt/presto-data

# 创建配置文件存放目录
mkdir /opt/trino-server-400/etc
```

分发到其他节点
```
# 由于我自己的集群机器是root免密登录的，所以用root用户分发，后面再修改权限分配
su root
cd  /opt/
scp -r /opt/trino-server-400 root@bigdata-node2:/opt/
scp -r /opt/trino-server-400 root@bigdata-node3:/opt/
scp -r /opt/presto-data root@bigdata-node2:/opt/
scp -r /opt/presto-data root@bigdata-node3:/opt/
```

### coordinator配置

在bigdata-node1上配置

- 配置 config.properties

\# 编辑文件
vim /opt/trino-server-400/etc/config.properties
```
# 新增以下内容
coordinator=true

#是否复用为worker节点，默认为 false。true代表coordinator节点还有worker节点的职能
node-scheduler.include-coordinator=false

# 本节点 presto 服务端口号
http-server.http.port=8089
query.max-memory=8GB
query.max-memory-per-node=1GB


#coordinator节点的ip和http服务端口
discovery.uri=http://bigdata-node1:8089
```

- 配置jvm.config

\# 编辑文件
vim /opt/trino-server-400/etc/jvm.config
```
# 新增以下内容
-server
-Xmx16G
-XX:-UseBiasedLocking
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+ExplicitGCInvokesConcurrent
-XX:+ExitOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError
-XX:ReservedCodeCacheSize=512M
-XX:PerMethodRecompilationCutoff=10000
-XX:PerBytecodeRecompilationCutoff=10000
-Djdk.attach.allowAttachSelf=true
-Djdk.nio.maxCachedBufferSize=2000000
```

- 配置node.properties

\# 编辑文件
vim /opt/trino-server-400/etc/node.properties
```
# 新增以下内容

#设置集群名称
node.environment=production

#节点唯一标识，同一个集群中的每个节点都不相同， 可以通过执行命令uuidgen获取, 也可以是计算机名
node.id=bigdata-node1

# 日志目录，计算临时存储目录， presto用户有读写权限
node.data-dir=/opt/presto-data/
```

### work配置

在`bigdata-node2` `bigdata-node3`上配置

- 配置 config.properties

\# 编辑文件
vim /opt/trino-server-400/etc/config.properties
```
# 新增以下内容
coordinator=false

#是否复用为worker节点，默认为 false。true代表coordinator节点还有worker节点的职能
node-scheduler.include-coordinator=true

# 本节点 presto 服务端口号
http-server.http.port=8089
query.max-memory=8GB
query.max-memory-per-node=1GB


#coordinator节点的ip和http服务端口
discovery.uri=http://bigdata-node1:8089
```

- 配置jvm.config

\# 编辑文件
vim /opt/trino-server-400/etc/jvm.config
```
# 新增以下内容
-server
-Xmx16G
-XX:-UseBiasedLocking
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+ExplicitGCInvokesConcurrent
-XX:+ExitOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError
-XX:ReservedCodeCacheSize=512M
-XX:PerMethodRecompilationCutoff=10000
-XX:PerBytecodeRecompilationCutoff=10000
-Djdk.attach.allowAttachSelf=true
-Djdk.nio.maxCachedBufferSize=2000000
```

- 配置node.properties

\# 编辑文件
vim /opt/trino-server-400/etc/node.properties
```
# 新增以下内容

#设置集群名称
node.environment=production

#节点唯一标识，同一个集群中的每个节点都不相同， 可以通过执行命令uuidgen获取, 也可以是计算机名
# bigdata-node2 => bigdata-node2
# bigdata-node3 => bigdata-node3
node.id=bigdata-node2

# 日志目录，计算临时存储目录， presto用户有读写权限
node.data-dir=/opt/presto-data/
```
### 配置日志级别（所有节点）

```
# 默认日志级别是INFO，如果需要改DEBUG模式就增加这个文件
vim /opt/trino-server-358/etc/log.properties

io.trino=DEBUG
```
默认的日志目录: /opt/presto-data/var/log

### 安装客户端（coordinator节点）

在coordinator节点，执行如下命令，安装客户端

```
#进入安装目录
cd /opt/trino-server-400/
 
#下载客户端
wget https://repo1.maven.org/maven2/io/trino/trino-cli/400/trino-cli-400-executable.jar
 
#重命名
mv trino-cli-400-executable.jar trino-cli
 
#赋予执行权限
chmod +x trino-cli

```

## 配置数据源（所有节点）

以mysql数据源为例

```
#创建并进入数据源目录
cd /opt/trino-server-400/etc/
mkdir catalog
cd catalog
 
#创建数据源文件
vim mysql.properties
 
#添加如下内容
connector.name=mysql
connection-url=jdbc:mysql://ip:3306
connection-user=user
connection-password=user
 
#设置catalog文件夹权限
chown -R presto:presto catalog
```

## 集群启动(所有节点)

```
chown -R presto:presto /opt/trino-server-355
chown -R presto:presto /opt/presto-data
```
>执行上面命令，主要为了防止配置的过程中，配置文件已root用户操作，在启动时presto用户权限不足

```
#在所有节点上执行
su presto
source ~/.bash_profile
# 检查java版本，确定是jdk17.0.3以上版本才有启动程序
java -version
cd /opt/trino-server-400
./bin/launcher start
./bin/launcher status
```
>Trino也提供了可视化的方式对集群状态和用户查询进行管理
>管理页面：http://bigdata-node1:8089
>默认用户名：admin，没有密码

## trino启动脚本制作（所有节点）

为了方便管理，避免以后启动trino忘记切换用户，制作了下面的脚本，一键启动

```
mkdir -p /home/manager-shell
vim /home/manager-shell/presto-start.sh
```

\# 添加下面的脚本

```
#!/bin/sh

if [ `whoami` != 'presto' ];then
        echo '要先切换到presto用户再启动presto服务，命令:su presto'
        exit 1
fi

echo "当前用户 = ${USER}"

source ~/.bash_profile

java -version

cd /opt/trino-server-400

./bin/launcher start
./bin/launcher status
```
>执行命令的方式： source ./presto-start.sh 或者 . ./presto-start.sh

# 资料
- [Presto(Trino)从安装到使用](https://blog.csdn.net/wang6733284/article/details/125431178)
- [CDH集成Trino(PrestoSQL)](https://blog.csdn.net/qaz1qaz1qaz2/article/details/119390420)
- [从 0 到 1 学习 Presto，这一篇就够了](https://cloud.tencent.com/developer/article/1892572)
- [认识Trino](https://blog.csdn.net/zsl2010zsl/article/details/124005540)
- [trino权限验证开启https](https://blog.csdn.net/w8998036/article/details/121699855)
- [初识Presto(Trino)](https://miaowenting.site/2021/03/04/%E5%88%9D%E8%AF%86Presto(Trino)/)