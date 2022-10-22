# CDH6.2.1部署flink 1.13

# 自制Flink Parcel集成CDH（Flink1.13.2 + CDH6.2.1+Scala2.11）

### 测试系统安装

为了方便，我本地测试环境一般都用Oracle VM VirtualBox来安装虚拟机，加上vagrant软件，可以通过几行代码配置的方式，一键启动各种虚拟机环境，不需要自己手动一个个系统安装。

### 安装软件

- Oracle VM VirtualBox： 最新版，官网下载就可以了
- [vagrant](https://www.vagrantup.com)：最新版，官网下载就可以了

安装之后，还需要修改一下这2个软件的缓存目录的环境变量，不然全部缓存放到c盘，迟早有一天会爆。

参考文献：

- [vagrant box保存路径修改](https://blog.csdn.net/t_1007/article/details/80048968)
- [VirtualBox指定虚拟机的存储位置](https://blog.csdn.net/sanqima/article/details/119651931)
- [vagrant 命令+配置+入门案例 - 快速创建 Centos7](https://www.cnblogs.com/lawsssscat/p/12676477.html)
- [官方镜像](https://vagrantcloud.com/boxes/search)

### 安装虚拟机

下文以windows为例，找一个位置，创建一个空文件夹，用来存放所有的vagrant box不同虚拟机的的配置文件，然后创建我们测试用虚拟机。

```
# 创建存放配置的文件夹
mkdir centos7

# 初始化配置文件
vagrant init centos7

# 启动虚拟机
vagrant up
```

测试环境已经创建好了，账户: vagrant， 密码: vagrant

修改root用户的登录密码命令

```
sudo -s passwd
```

关于软件的使用，参考上面的文件。

### 注意事项

由于官方的generic/centos7镜像默认是不开启ssh，需要自己在VirtualBox软件上面，进入虚拟机先修改才能用ssh登录。


## 环境安装

### 安装JDK1.8

为了保持和CDH安装的时候那个jdk环境一样，选用一模一样的安装包，下载地址：[oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm](https://archive.cloudera.com/cm6/6.2.1/redhat7/yum/RPMS/x86_64/oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm)

把文件上传到/home/tools目录下


安装jdk

```
[root@centos7 tools]# rpm -ivh oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
```
>注意：jdk默认会被安装到/usr/java目录中。

配置jdk环境变量，在/etc/profile文件末尾增加下面内容
```
[root@centos7 tools]# vi /etc/profile
#.....
export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
export PATH=.:$PATH:$JAVA_HOME/bin
```

重新加载环境变量
```
[root@centos7 tools]# source /etc/profile
```

验证jdk是否安装成功
```
[root@centos7 tools]# java -version
java version "1.8.0_181"
Java(TM) SE Runtime Environment (build 1.8.0_181-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.181-b13, mixed mode)
```

### 安装Maven

Maven 下载
Maven 下载地址：http://maven.apache.org/download.cgi

下载并解压
```
# wget https://dlcdn.apache.org/maven/maven-3/3.8.6/binaries/apache-maven-3.8.6-bin.tar.gz
# tar -xvf  apache-maven-3.8.6-bin.tar.gz
# sudo mv -f apache-maven-3.8.6 /usr/local/
```

编辑 /etc/profile 文件 sudo vim /etc/profile，在文件末尾添加如下代码：
```
export MAVEN_HOME=/usr/local/apache-maven-3.8.6
export PATH=${PATH}:${MAVEN_HOME}/bin
```

保存文件，并运行如下命令使环境变量生效：
```
# source /etc/profile
```

在控制台输入如下命令，如果能看到 Maven 相关版本信息，则说明 Maven 已经安装成功：
```
# mvn -v
```

### 安装cm_ext

Cloudera提供的cm_ext工具，对生成的csd和parcel进行检验

```
mkdir -p ~/github/cloudera
cd ~/github/cloudera

# clone cm_ext
git clone https://github.com/cloudera/cm_ext.git

# 打包
cd cm_ext
mvn package -Dmaven.test.skip=true
```
>Tips：build_parcel.sh和 build_csd.sh脚本文件里面执行jar包路径默认是~/github/cloudera/

# 制作parcel

```
parcel制作方式大致有两种，第一种是使用源生的制作方法，制作过程繁琐复杂，第二种是使用广大网友制作好的parcel制作工具，本文使用后者

先创建临时文件夹，如果跟着上文操作的可以跳过创建文件夹
mkdir -p ~/github/cloudera
cd ~/github/cloudera

下载制作工具：
git clone https://github.com/pkeropen/flink-parcel.git
完成后会在当前目录生成一个flink-parcel的文件，证明下载成功

修改配置文件
cd ./flink-parce
vim flink-parcel.properties
进行相应修改，内容如下：
#FLINK 下载地址
FLINK_URL=https://archive.apache.org/dist/flink/flink-1.13.2/flink-1.13.2-bin-scala_2.11.tgz

#flink版本号
FLINK_VERSION=1.13.2

#扩展版本号
EXTENS_VERSION=BIN-SCALA_2.11

#操作系统版本，以centos为例
OS_VERSION=7

#CDH 小版本
CDH_MIN_FULL=5.2
CDH_MAX_FULL=6.3.3

#CDH大版本
CDH_MIN=5
CDH_MAX=6


保存并退出

然后进行build
./build.sh  parcel
下载并打包完成后会在当前目录生成FLINK-1.13.2-BIN-SCALA_2.11_build文件

构建flink-yarn csd包
./build.sh csd_on_yarn
执行完成后会生成FLINK_ON_YARN-1.13.2.jar

将FLINK-1.13.2-BIN-SCALA_2.11_build打包
tar -cvf ./FLINK-1.13.2-BIN-SCALA_2.11.tar ./FLINK-1.13.2-BIN-SCALA_2.11_build/

将FLINK-1.13.2-BIN-SCALA_2.11.tar 
FLINK_ON_YARN-1.12.0.jar下载，这两个包就是目标包

上传到正式环境服务器(局域网yum提供的节点)
```

# 集成cm

把下载好的FLINK-1.13.2-BIN-SCALA_2.11.tar , FLINK_ON_YARN-1.12.0.jar放到自己的apache www目录下面，和安装cdh一样的，如果有NAS的可以开启webdav服务，更加方便了。

解压FLINK-1.13.2-BIN-SCALA_2.11.tar文件，得到FLINK-1.13.2-BIN-SCALA_2.11_build文件夹

```
# 每个节点操作一次，把parcel安装包下载到节点上
mv FLINK-1.13.2-BIN-SCALA_2.11-el7.parcel /opt/cloudera/parcel-repo/
mv FLINK-1.13.2-BIN-SCALA_2.11-el7.parcel /opt/cloudera/parcel-repo/

#登录服务器，将FLINK_ON_YARN-1.13.2.jar上传到cm主节点的/opt/cloudera/csd/目录下（目的是让cm识别）
mv FLINK_ON_YARN-1.13.2.jar /opt/cloudera/csd/

重启server和agent
systemctl restart cloudera-scm-server
systemctl restart cloudera-scm-agent

登录cm
在parcel配置界面添加flink的parcel源
然后进行下载→分配→解压→激活

然后可以开始安装了
```

# 可能会出现的问题

## 缺少jar包

```
缺yarn的依赖和缺少flink连接hadoop3的包还有一个hadoop-common-3.0.0-cdh6.2.1.jar commons-cli-1.4.jar flink-shaded-hadoop-3-uber-3.1.1.7.2.9.0-173-9.0.jar(所有flink节点都需要添加)，可以自行下载这两个包（联系博主也可以）

mv hadoop-common-3.0.0-cdh6.2.1.jar commons-cli-1.4.jar flink-shaded-hadoop-3-uber-3.1.1.7.2.9.0-173-9.0.jar /opt/cloudera/parcels/FLINK/lib/flink/lib/

添加完成后再重试还会报一个与Kerberos相关的错误，由于我的集群并没有开启Kerberos，所以需要到flink的配置界面中把Kerberos相关的配置删除，完后再重启就能够正常启动了。

如果启动有问题，多查看/var/log/flink/下面的日志
```

## 8081端口冲突导致启动失败

如果你和我一样是3节点的，检查一下CM后台flink的配置，把yarn.taskmanagers改成2（总节点数-1），因为默认值是1，导致多出来的那个节点要抢rest.port服务端口导致其中有一台机启动失败。

# 资料

- [自制Flink Parcel集成CDH（Flink1.13.2 + CDH6.2.1+Scala2.11）](https://blog.csdn.net/weixin_44389063/article/details/119579087)
- [CDH6.2.1集成flink1.9.0](https://blog.csdn.net/weixin_43704599/article/details/106593602)
- https://www.modb.pro/db/133009
- https://www.programminghunter.com/article/51042461516/
- https://blog.csdn.net/spark9527/article/details/115767011