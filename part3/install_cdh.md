# CDH大数据环境安装


# 资料

- [CDH6.2大数据平台安装部署](http://wiki.xuwei.tech/hadoop1/cdh.html)


# 避坑

## 安装过程

上面这个教程比较完善了，基本上可以快速安装好CDH，由于我之前看过很多文章，下载过3个CDH的版本：`CDH6.2.0`，`CDH6.2.1`，`CDH6.3.2`，当时按照上面教程，跳过CDH包那步，想一次性安装`CDH6.3.2`，结果界面没有显示，后来先安装`CDH6.3.2`，然后再升级上去。

另外很多软件包已经快失传了，直接下载官方的时不可能下载到的，建议先迅雷下载，迅雷有镜像包，还可以提供资源下载。

教程中`安装Httpd服务`这一步，因为我自己有NAS，直接在NAS上面做`webdav`下载功能的，后来思考了一下，不建议大家跟教程在cdh01节点上面做，因为在后面更新或者安装其他软件的时候经常会用到，建议有条件的就在自己NAS上面建相关的下载服务，如果没有NAS条件的，建议另外建一个虚拟机或者云服务器提供服务，长期用的。

## 快速初始化新系统脚本

初始化要做的事情：
- 更换国内源
- 更新软件，centos7.9其实很老的了，初始化系统最好更新一下
- 设置时区（记得更换）
- 设置静态ip
- 关闭firewalld
- 设置主机名
- 关闭selinux
- 改变swap大小 (建议改成2G或者关闭，特别是有打算装doris的)
- 添加用户 （可选，可后面操作，特别是装spark，要添加新个人用户，不能用root进行spark-submit，会失败）


快速脚本
- [centos7 快速初始化脚本](https://github.com/bboysoulcn/centos.git)
- [一键更换国内软件源脚本](https://gitee.com/SuperManito/LinuxMirrors)

## mysql版本问题

因为之前在NAS上面部署了一个mysql8和redash查询工具服务，然后手多更换了教程上面的mysql版本，从5.7缓存8，结果哭惨了，数据不能导入，要删除了，重新安装。查资料发现CM管理工具不支持mysql8。

## NTP时间服务器问题

踩坑最多的地方，参考我`集群时间戳不一致导致的问题及其分析和思考`这个文章

NTP时间服务没有做好，后期会在CM管理系统是不是报服务不健康的错误，如果用doris，doris还要要求5秒内的时差，如果一开始没有做好，后期只能泪流满面。