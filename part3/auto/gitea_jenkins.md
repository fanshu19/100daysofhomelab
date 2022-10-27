# gitea + Jenkins自动化部署项目

# gitea 和 gitlab选型

gitea 只是 git 的操作库实现搭配了基础 wiki 这些功能，需要配合其他第三方库或工具才能提供更专业的支持。比如 cicd 要 drone。Gitea 功能比较简单，看你们的需求如果只是单纯存存代码，只做 CI/CD 是足够的，很多附加功能处于缺失状态，需要大量依赖第三方工具。

Gitlab 是完整的 git 集成环境，包含 npm，nuget，docker registry 等私有集成，还有完整的 CI/CD，k8s 集成方案。Gitlab 需要的机器最低配置比较高，对应的，功能也多了很多，如果需要代码 Review 、重度使用 issue 功能、使用 Gitlab 管理项目就比较合适。

决定性因素：对于计算机资源的占用，gitlab明显就是一直超级怪兽，而我的truenas系统只有32g，选择小型化又满足基本功能的gitea就可以了。

# Jenkins自动化部署

好处：自动化，开发者push代码之后，全自动编译，打包项目文件上传到服务器指定位置，避免人为出错。
坏处：看项目大小，这个还是比较占用计算机资源，限制好内存大小。花点时间学习一下。

## Jenkins发布到服务器的时候设置

说明：

source files 就是准备上传的文件，该文件是相对于这个项目的workspace目录，也就是$JENKINS_HOME/workspace/xxxx/ 每个项目都会在build时候自动创建worksapce

比如我要上传

$JENKINS_HOME/workspace/xxxx/target/class/helloworld1.java

$JENKINS_HOME/workspace/xxxx/target/class/helloworld2.java 

那么我们就可以设置如下参数

source files=target/class/*.java

remove prefix = target (remove prefix必须是source files中指定的目录，如果不写，那就是把这个目录层级都上传，如果写target，就传class目录层级，如果写target/class 就传*.java文件)

remote diretory = rd (remote diretory就是相对于系统配置中对服务器配置中的remote diretory来说的，比如在服务器配置中的remote diretory如果是空，那应该就是家目录，如果不是空，假如是/usr/local)

那这样上传过去，文件存在服务器的目录是   /usr/local/rd/class/*.java

也就是   服务器配置里的remote diretory[/usr/local]+这里配置的remote diretory[rd]+source files去掉remove prefix的目录剩下的部分[class/*.java]



# 资料

- https://blog.csdn.net/tianmingqing0806/article/details/125408239
- https://www.cnblogs.com/yxhblogs/p/14056752.html
- https://codeantenna.com/a/wcYUhntuqP
- https://juejin.cn/post/6844903813120262158