# 编写生产级 PySpark 作业的最佳实践

# 资料

- [Best Practices Writing Production-Grade PySpark Jobs](https://developerzen.com/best-practices-writing-production-grade-pyspark-jobs-cb688ac4d20f#.wg3iv4kie)
- [ekampf/PySpark-Boilerplate](https://github.com/ekampf/PySpark-Boilerplate.git)

# 说明

详细的教程就不说了，看英文文档。目前已经运行了几年，团队开发和维护都比较方便。

目录：

`jobs`文件夹，可以多级目录，调用的时候，参数里面`--job`里面写的值就例如这样`test.wordcount`，标示不同级别的目录。

# 编译上线

```
# 安装项目所依赖的包
pip install -U -r requirements.txt -t ./src/libs

# 开始打包上线，最后就是`main.py`, `jobs.zip`, `libs.zip`放到服务器上面就可以运行了。
mkdir ./dist
cp ./src/main.py ./dist
cd ./src && zip -x main.py -x \*libs\* -r ../dist/jobs.zip .
cd ./src/libs && zip -r ../../dist/libs.zip .
```