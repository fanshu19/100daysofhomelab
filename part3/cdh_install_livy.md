# CDH6集成Livy服务 HUE支持sparksql查询

# 资料

- [CDH6集成Livy服务](http://www.yujc.online/article/cdh_livy_01/)
- [CDH使用CM安装整合livy和zepplin(已攻略)](https://blog.csdn.net/qq_36428889/article/details/119778919)
- [Apache Livy 安装部署使用示例](https://blog.csdn.net/qq_43081842/article/details/123822382)

# 编译注意

> 超级注意：要编译`livy`的`0.7.0`以上版本，如果编译`0.5.0`版本，会出现使用spark-sql的时候，decimal数据类型出错，是scale版本不支持导致的。可以查看[官网更新日志](https://livy.apache.org/history/)，是因为0.7.0以上才支持spark `2.4.x`。

教程里面是先编译，后解压parcel文件，修改meta/parcel.json，
其实可以一开始就编辑livy-parcel-src/meta/parcel.json里面，添加添加livy用户和组，然后再编译就不需要解压，后编辑。

cdh 6.3.2版本重启
```
systemctl restart cloudera-scm-server
```
重启是上面的命令，不是在后台界面操作