# dolphinscheduler

# 为什么选择dolphinscheduler

想比较详细了解几个ETL系统的对比，可以参考[我和Apache DolphinScheduler的缘分](https://blog.csdn.net/dolphinscheduler/article/details/112174473)。我说一下我在实际工作中用azkaban和dolphinscheduler的直观感受。

在公司几年，数据增量也是每年暴增几倍的速度增长，从最开始的mysql，到早期开始转大数据技术，再到后来的技术成熟，期间接触过2套ETL系统，一个是azkaban，一个dolphinscheduler，ETL系统背后也是代表不同的技术升级。早期的大数据技术是用azkaban来完成任务调度的，用kettle软件编写并发布，我只能说我是超级超级讨厌这套系统的，我们最初是用这个系统编写各种sql来统计，还是可以完成基础任务的，但是到了后期，如果需要对数据进行进一步处理，kettle就变得难以完成；再者就是它没有DAG和资源监控功能，这个在开发过程中排查问题超级困难，浪费大量的时间，也不清楚员工开发的功能会浪费多少服务器资源，只能看个总数，不利于日后清理任务时候统计一些暂用资源打的服务。最后也是特别让我们团队诟病而且厌恶的就是变量注入，因为ETL系统避免不来的就是重跑数据，新功能批量跑数据，azkaban没有这样的支持，导致整个跑批任务要全程人工干预，效率超级低。

后来技术升级到spark，用pyspark实现统计，跑批处理之后，ETL系统也改用到dolphinscheduler，基本上可以说是全方面支持各种我们需要的功能，特别是批量跑数据，加入参数之后，结合shell，这些操作变得超级简单，可以完全挂机跑任务，员工也可以不需要加班。加上yarn调度，spark的DAG和日志功能，排查问题也简单了好多。

# 安装

- [官方教程](https://dolphinscheduler.apache.org/zh-cn/docs/1.3.9/user_doc/cluster-deployment.html)

- 安装教程：
    - https://blog.csdn.net/qq_42502354/article/details/116537022
    - https://juejin.cn/post/6953261989376294948

- [使用教程](https://www.cnblogs.com/ttzzyy/p/13394488.html)

# 遇到的坑

- 版本选择，dolphinscheduler已经到了3.0版本了，但是看官方文档，新版本对系统的mysql、mysql-connector.jar、zookeeper等版本要求都比较高，CDH6.3.2不支持这么高，所以选择了1.3.9版本，这个版本其实已经满足超级多需求的了。

- dolphinscheduler要添加到supergroup用户组里面，所有节点都要，否则因为hdfs权限问题，启动失败

- pyspark项目要在每个节点相同位置都要有，否则启动任务找不到文件。配置Jenkins打包完之后都上传到每一个节点上面去

# 使用经验

- 在`资源中心`的`文件管理`创建公共shell文件，统一管理spark启动任务的脚本，以后任务一多，修改配置也方便

- 任务管理规范要做好，用表名，ETL主题名字等等命名，把多个任务放到`工作流`里面，不然到了后期有上百个任务的时候，找都找不到在哪里`工作流`里面。