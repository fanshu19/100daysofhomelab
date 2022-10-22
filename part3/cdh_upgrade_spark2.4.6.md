# CDH6.2.1 升级 spark2.4.6版本

# 背景

由于 CDH6.3.2 以上，已不开源。常用组件只能自编译升级，比如 Spark 。这次决定升级spark，但是不选择3.x版本主要是因为公司用的最新版本也就2.4.4，而且升级最新版的话，其他附带的服务也要跟着升级，动作太大，不适宜已经在生产环境部署。

# 编译环境

| 名称 | 版本 | 备注 |
|  :----  | :----  | :----  |
| jdk  | 1.8.0_181 | oracle的 |
| Maven  | 3.8.4+ | 直接安装最新版就可以了 |
| Centos  | 7.9 |  |

我建议本地电脑装虚拟机编译，cpu和内存都够大，不要再线上环境或者云服务器上编译。编译环境部署可以参考我[CDH6.2.1部署flink 1.13](cdh_deploy_flink.md)文章里面的环境部署内容

# 下载软件

下载需要编译的软件，我存放目录是/home/vagrant/tools

```
wget https://archive.apache.org/dist/spark/spark-2.4.6/spark-2.4.6.tgz

# 创建临时编译用的目录
mkdir spark
# 解压源代码
tar -zxvf spark-2.4.6.tgz -C spark
```

# 编译spark

## 修改make-distribution.sh

```
# 进入源代码目录
cd spark/spark-2.4.6

#修改版本号为固定版本,避免编译时脚本自动获取
vim dev/make-distribution.sh
```

找到下面那段代码，并注释掉，然后添加新代码

```
VERSION=$("$MVN" help:evaluate -Dexpression=project.version $@ 2>/dev/null\
    | grep -v "INFO"\
    | grep -v "WARNING"\
    | tail -n 1)
SCALA_VERSION=$("$MVN" help:evaluate -Dexpression=scala.binary.version $@ 2>/dev/null\
    | grep -v "INFO"\
    | grep -v "WARNING"\
    | tail -n 1)
SPARK_HADOOP_VERSION=$("$MVN" help:evaluate -Dexpression=hadoop.version $@ 2>/dev/null\
    | grep -v "INFO"\
    | grep -v "WARNING"\
    | tail -n 1)
SPARK_HIVE=$("$MVN" help:evaluate -Dexpression=project.activeProfiles -pl sql/hive $@ 2>/dev/null\
    | grep -v "INFO"\
    | grep -v "WARNING"\
    | fgrep --count "<id>hive</id>";\
    # Reset exit status to 0, otherwise the script stops here if the last grep finds nothing\
    # because we use "set -o pipefail"
    echo -n)
```

最终修改效果
```
#VERSION=$("$MVN" help:evaluate -Dexpression=project.version $@ 2>/dev/null\
#    | grep -v "INFO"\
#    | grep -v "WARNING"\
#    | tail -n 1)
#SCALA_VERSION=$("$MVN" help:evaluate -Dexpression=scala.binary.version $@ 2>/dev/null\
#    | grep -v "INFO"\
#    | grep -v "WARNING"\
#    | tail -n 1)
#SPARK_HADOOP_VERSION=$("$MVN" help:evaluate -Dexpression=hadoop.version $@ 2>/dev/null\
#    | grep -v "INFO"\
#    | grep -v "WARNING"\
#    | tail -n 1)
#SPARK_HIVE=$("$MVN" help:evaluate -Dexpression=project.activeProfiles -pl sql/hive $@ 2>/dev/null\
#    | grep -v "INFO"\
#    | grep -v "WARNING"\
#    | fgrep --count "<id>hive</id>";\
#    # Reset exit status to 0, otherwise the script stops here if the last grep finds nothing\
#    # because we use "set -o pipefail"
#    echo -n)

# 添加下面代码
VERSION=2.4.6
SCALA_VERSION=2.11
SPARK_HADOOP_VERSION=3.0.0-cdh6.2.1
SPARK_HIVE=1 # 支持spark on hive

```

说明：

`VERSION`：spark的版本号，修改成你当前下载的版本

`SCALA_VERSION`：scala版本，2.4.x系列可用版本是2.11或者2.12，具体版本查一下资料

`SPARK_HADOOP_VERSION`：hadoop的版本，看一下你自己当前的版本，我用的是cdh，所以这个版本

修改maven编译内存，加快速度

```
#找到下面的代码
export MAVEN_OPTS="${MAVEN_OPTS:--Xmx2g -XX:ReservedCodeCacheSize=512m}"

然后修改成自己可以接受的内存
export MAVEN_OPTS="${MAVEN_OPTS:--Xmx8g -XX:ReservedCodeCacheSize=2g}"
```

## 修改pox.xml

修改pox.xml文件 （在源代码的根目录），增加 cloudera maven 仓库

在 repositories 标签下，新增

```
<repository>
    <id>cloudera</id>
    <url>https://repository.cloudera.com/artifactory/cloudera-repos</url>
    <releases>
        <enabled>true</enabled>
    </releases>
    <snapshots>
        <enabled>false</enabled>
    </snapshots>
</repository>
```

如果你没有上外网的基础，需要多增加下面这个国内仓库
```
<repository>
    <id>aliyun</id>
    <url>https://maven.aliyun.com/nexus/content/groups/public</url>
    <releases>
        <enabled>true</enabled>
    </releases>
    <snapshots>
        <enabled>false</enabled>
    </snapshots>
</repository>
```

## 开始编译

```
./dev/make-distribution.sh \
--name hadoop3.0.0-cdh6.2.1 --tgz  -Pyarn -Phadoop-3.0 \
-Phive-thriftserver -Dhadoop.version=3.0.0-cdh6.2.1 -DskipTests  -X
```
> `-X`这个参数是debug模式，会输出更多日志，如果不想看的话，把这个参数删除； `-Dhadoop.version`这个参数是对应你hadoop的版本

经过1小时漫长等待，终于编译好了，文件spark-2.4.6-bin-hadoop3.0.0-cdh6.2.1.tgz

# 部署到集群

上传编译好的文件到要部署 spark3 的客户端机器上

```
tar -zxvf spark-2.4.6-bin-hadoop3.0.0-cdh6.2.1.tgz -C /opt/cloudera/parcels/CDH/lib
cd /opt/cloudera/parcels/CDH/lib
mv spark-2.4.6-bin-hadoop3.0.0-cdh6.2.1/ spark2
```

## 添加CDH环境配置

链接hadoop/hive相关的配置文件到 conf目录下

```
cd spark2

ln -s /etc/hive/conf/hdfs-site.xml hdfs-site.xml
ln -s /etc/hive/conf/mapred-site.xml mapred-site.xml
ln -s /etc/hive/conf/yarn-site.xml yarn-site.xml
ln -s /etc/hive/conf/core-site.xml core-site.xml
ln -s /etc/hive/conf/hive-env.sh hive-env.sh
ln -s /etc/hive/conf/hive-site.xml hive-site.xml
```

添加spark配置

```
cp /etc/spark/conf/spark-env.sh  /opt/cloudera/parcels/CDH/lib/spark2/conf
chmod +x /opt/cloudera/parcels/CDH/lib/spark2/conf/spark-env.sh

#修改 spark-env.sh
vim /opt/cloudera/parcels/CDH/lib/spark2/conf/spark-env.sh

export SPARK_HOME=/opt/cloudera/parcels/CDH/lib/spark3

cp /etc/spark/conf/log4j2.properties  /opt/cloudera/parcels/CDH/lib/spark2/conf
cp /etc/spark/conf/spark-defaults.conf  /opt/cloudera/parcels/CDH/lib/spark2/conf

# 修改spark-defaults.conf，下面写的修改，其他的保持不变
# 修改结果
spark.yarn.jars=local:/opt/cloudera/parcels/CDH-6.2.1-1.cdh6.2.1.p0.1425774/lib/spark2/jars/*,local:/opt/cloudera/parcels/CDH-6.2.1-1.cdh6.2.1.p0.1425774/lib/spark2/hive/*
#spark.extraListeners=com.cloudera.spark.lineage.NavigatorAppListener
#spark.sql.queryExecutionListeners=com.cloudera.spark.lineage.NavigatorQueryListener
```

## 创建 spark-sql

```
vim /opt/cloudera/parcels/CDH/bin/spark-sql

#!/bin/bash 
export HADOOP_CONF_DIR=/etc/hadoop/conf
export YARN_CONF_DIR=/etc/hadoop/conf
SOURCE="${BASH_SOURCE[0]}"  
BIN_DIR="$( dirname "$SOURCE" )"  
while [ -h "$SOURCE" ]  
do  
 SOURCE="$(readlink "$SOURCE")"  
 [[ $SOURCE != /* ]] && SOURCE="$BIN_DIR/$SOURCE"  
 BIN_DIR="$( cd -P "$( dirname "$SOURCE" )" && pwd )"  
done  
BIN_DIR="$( cd -P "$( dirname "$SOURCE" )" && pwd )"  
LIB_DIR=$BIN_DIR/../lib  
export HADOOP_HOME=$LIB_DIR/hadoop  
  
# Autodetect JAVA_HOME if not defined  
. $LIB_DIR/bigtop-utils/bigtop-detect-javahome  
  
exec $LIB_DIR/spark2/bin/spark-submit --class org.apache.spark.sql.hive.thriftserver.SparkSQLCLIDriver "$@"
```

配置 spark-sql 快捷方式

```
chmod +x /opt/cloudera/parcels/CDH/bin/spark-sql
alternatives --install /usr/bin/spark-sql spark-sql /opt/cloudera/parcels/CDH/bin/spark-sql 1
```

## 创建 spark3-submit

```
vim /opt/cloudera/parcels/CDH/bin/spark2-submit

#!/bin/bash
  # Reference: http://stackoverflow.com/questions/59895/can-a-bash-script-tell-what-directory-its-stored-in
  SOURCE="${BASH_SOURCE[0]}"
  BIN_DIR="$( dirname "$SOURCE" )"
  while [ -h "$SOURCE" ]
  do
    SOURCE="$(readlink "$SOURCE")"
    [[ $SOURCE != /* ]] && SOURCE="$DIR/$SOURCE"
    BIN_DIR="$( cd -P "$( dirname "$SOURCE"  )" && pwd )"
  done
  BIN_DIR="$( cd -P "$( dirname "$SOURCE" )" && pwd )"
  LIB_DIR=$BIN_DIR/../lib
export HADOOP_HOME=$LIB_DIR/hadoop

# Autodetect JAVA_HOME if not defined
. $LIB_DIR/bigtop-utils/bigtop-detect-javahome

# disable randomized hash for string in Python 3.3+
export PYTHONHASHSEED=0

exec $LIB_DIR/spark2/bin/spark-submit "$@"

```

配置 spark2-submit 快捷方式

```
chmod +x /opt/cloudera/parcels/CDH/bin/spark3-submit
alternatives --install /usr/bin/spark2-submit spark2-submit /opt/cloudera/parcels/CDH/bin/spark2-submit 1
```

# 分发spark软件包

```
scp -r spark2 root@bigdata-node2:/opt/cloudera/parcels/CDH/lib/
scp -r spark2 root@bigdata-node3:/opt/cloudera/parcels/CDH/lib/
```

# 测试

```
spark2-submit --class org.apache.spark.examples.SparkPi --master yarn --deploy-mode cluster --driver-memory 4g --executor-memory 2g --executor-cores 1 /opt/cloudera/parcels/CDH/lib/spark2/examples/jars/spark-examples*.jar 10
```

# 资料

- [spark2.4.4-CDH6.3.0编译](https://www.jianshu.com/p/4deb46d2d144)
- [Spark SQL 3.0.1 与 CDH Hive 2.1.1结合](https://blog.csdn.net/Young2018/article/details/108871542)
- [CDH6.3.2 升级 Spark3.3.0 版本](https://mp.weixin.qq.com/s/H8vr8EMKkCSAta-F1ecVXA)
- 错误问题修改 ：https://blog.csdn.net/qq_26442553/article/details/126028533
- https://developer.aliyun.com/article/232648
- https://codeantenna.com/a/Uuz3Z8pDkc
- https://hijiazz.gitee.io/apache-spark247-hadoop31-build/
- https://blog.csdn.net/weixin_44641024/article/details/102624086
- https://blog.csdn.net/qq_43591172/article/details/126575084
- [cdh6.x 集成spark-sql](https://blog.csdn.net/qq_26442553/article/details/126028533)
- https://blog.csdn.net/qq_26502245/article/details/120355741