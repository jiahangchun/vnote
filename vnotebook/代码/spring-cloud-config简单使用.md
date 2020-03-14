# spring cloud config 简单使用

[toc]

## 痛点

* convenient项目每次合并代码，总会出现application-local.yml冲突
* convenient项目切个环境还要我去找ip和密码么？！
* 我总是不小心将application-local.yml提交到new-master怎么办？我又不是故意的
* 之前尝试过githook ,但是对于每来一个需求删一个工程的人来说，还是太麻烦了。
* 行吧，我自己本地保存一个行了吧。但是为什么每次都有人不加载application.yml里面？
* 有些项目，用本来的配置还跑不起来！
* 相比较而言，少了许多配置文件，看上去顺眼多了。



## 准备工作

Spring cloud config 的引入





## 使用指南

* 配置工程 [config项目](http://git.jimuitech.com/hangchun.jia/config.git):test分支

* convenient项目:test-jhc-200308-testConfig分支

* 注册中心用现有的测试环境的eureka

  > 对于eureka要单独配置，其实我也有自己的想法：为什么不能直接在dockerFile里面直接通过 java -jar Xxx.jar --eureka.ip=xxxx 这样的形式呢？反正如果使用了config，那配置将大大减小



## 比较

| name                | 缺点                                                         | 优点                                                         |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Apollo              | 一听就是个庞然大物，<br />之前分享不就有个关于apollo么？<br />听着就比较麻烦 | 功能比较全面吧<br />现在我们项目里面支持了吧                 |
| discon              | 去看了下，没想到又要安装mysql.....一系列东西，<br />太麻烦了吧。 |                                                              |
| nacos               | 这个要配合nacos作为注册中心使用吧                            |                                                              |
| spring cloud config | 它的文档和sample真是少得可怜<br /><br />感觉很多东西都没讲清楚 | 1/我相信这个比Apollo简单很多了吧<br />2/万一apollo挂了，都不知道怎么处理。<br />但是config呢？直接使用eureka做高可用的，只要看下有没有注册就可以了<br />3/每个人都能到那个git上面定义属于自己的配置<br />4/最关键的是：我有现成的demo啊，只要像普通工程那样启动下就行了。<br />代码也简单，搞不定了，自己可以看代码<br />5/apollo 有 自动更新，config也是有的（虽然demo没有写），主要是用githook & spring cloud bus |









## 注意事项

* 安全性：我们工程都建在docker里面了，还需要？
  实在要的话 可以用spring-security or keystroke  ，它那个demo里面有的
* 需要 改脚本了
* 这个很依赖管理员维护的，万一一些环境崩了，有点麻烦的
* 当然了，实在不行就自定义application.yml。但是这和我的初衷是违背的。
* 对了 还有一些 list 格式的 配置是不支持的。但是我看过了，有些项目里面其实早不用了，可以删除了。如果真想要，其实我也有办法：直接写在application.yml里面 或者 数据库配置 或者 说 用grooxy外部排除掉（这样我们还能随便改规则）



## 效果图



![实际convenient项目效果图](/Users/jiahangchun/Desktop/实际convenient项目效果图.png)



![网上查看配置](/Users/jiahangchun/Desktop/网上查看配置.png)





## 参考

[spring cloud config 详细使用](https://www.cnblogs.com/fengzheng/p/11242128.html )

[官方demo](https://spring.io/projects/spring-cloud-config)

[spring cloud config 简介](https://www.cnblogs.com/fengzheng/p/11242128.html)

[最佳实践](https://www.cnblogs.com/xiaoliu66007/p/8963934.html)

[spring配置文件加载顺序问题](