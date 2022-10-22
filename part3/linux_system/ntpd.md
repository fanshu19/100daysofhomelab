# 集群时间戳不一致导致的问题及其分析和思考

# ntp
对Linux而言，可以使用ntp来实现集群的时间同步。

同步时间，可以使用ntpdate命令，也可以使用ntpd服务。

# ntpdate
使用`ntpdate`比较简单。格式如下：

```
$ ntpdate [-nv] [NTP IP/hostname]
$ ntpdate 192.168.0.1
$ ntpdate time.ntp.org
```
`ntpdate`只是强制将系统时间设置为ntp服务器时间。如果机器cpu tick有问题，这种同步治标不治本。此时，一般配合cron任务来定期同步。例如：

```
3 * * * * /usr/sbin/ntpdate 172.17.1.1
```

# ntpd

使用`ntpd`服务，要好于使用`ntpdate+cron`。

`ntpdate`同步时间，会造成时间的跳跃，对一些依赖时间的程序和服务会造成影响，比如sleep，timer等。

`ntpd`服务可以在修正时间的同时，修正cpu tick。

理想的做法为，在开机的时候，使用`ntpdate`强制同步时间，在其他时候使用`ntpd`服务来同步时间。

需要注意的是，`ntpd`有一个自我保护设置: 如果本机时间与源时间相差太大, 则`ntpd`不运行。所以新设置的时间服务器一定要先利用`ntpdate`命令获取初始时间, 然后启动`ntpd`服务。

关于`ntpd`的使用，网上已经有很多资料了，这里就不做过多的介绍了。

# ntpstat

`ntpstat`用于显示网络时间的同步状态。`ntpstat`会给出本地计算机上运行的`ntpd`的同步状态。如果本地系统与参考时间源同步，则`ntpstat`会给出近似的时间精度。

```
$ ntpstat
synchronised to NTP server (192.168.0.1) at stratum 3
   time correct to within 43 ms
   polling server every 1024 s
```
   
问题引申
虽然可以利用ntp来同步集群时间，但是有一个问题是值的思考的：如何在集群时钟不同的情况下，解决时间戳不一致带来的问题一节中描述的问题。

系统在架构上要做何种优化，才能避免业务对集群时钟同步的依赖？而这种优化所带来的系统架构的复杂度和成本又将如何？

# 资料

- [集群时间戳不一致导致的问题及其分析和思考](https://wangwei1237.github.io/2020/05/29/problem-caused-by-timestamp-s-difference-in-cluster-and-analysis/)
- [那些年我们忽略的集群时钟不同步问题](https://blog.csdn.net/showchi/article/details/106230110)