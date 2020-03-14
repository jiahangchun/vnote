# spring-cloud-config 简单了解


[toc]

## 结构

### **ConfigServerConfiguration**

|                 类名                 |                                                                简述                                                                |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| EnvironmentRepositoryConfiguration  | 环境变量存储相关的配置类                                                                                                              |
| CompositeConfiguration              | 组合方式的环境仓库配置类                                                                                                              |
| ResourceRepositoryConfiguration     | 资源仓库相关的配置类                                                                                                                 |
| ConfigServerEncryptionConfiguration | 加密断点相关的配置类                                                                                                                 |
| ConfigServerMvcConfiguration        | 对外暴露的MVC端点控制器的配置类                                                                                                       |
| TransportConfiguration              | ```Configure a callback to set up a property based SSH settings before running a transport command (such as clone or fetch)``` |

### **EnvironmentRepositoryConfiguration**(这里应该就是我要的实现了)

| 类名                           | 简述                                                         |
| ------------------------------ | :----------------------------------------------------------- |
| JdbcRepositoryConfiguration    | 嘿嘿，这个不就是我们的basic_global_configure么？             |
| VaultRepositoryConfiguration   | 这个没看懂，是从请求里面拿么？                               |
| SvnRepositoryConfiguration     | Svn                                                          |
| NativeRepositoryConfiguration  | ```Simple implementation of {@link EnvironmentRepository} that uses a SpringApplication * and configuration files located through the normal protocols. The resulting Environment * is composed of property sources located using the application name as the config file * stem (spring.config.name) and the environment name as a Spring profile.``` |
| GitRepositoryConfiguration     | 这个就是我想找的方法吧                                       |
| DefaultRepositoryConfiguration | 默认实现，而 GitRepositoryConfiguration extends DefaultRepositoryConfiguration |



### GitRepositoryConfiguration实现的时序图

Git 其实没有单独的实现，它的实现都放在了default 的策略里面

``` flowchart
title: 具体实现的时序图
participant DefaultRepositoryConfiguration as A
participant Spring容器 as B
participant ConfigServer as C
Note over A:通过spring配置收集需要的信息
Note left of A: ConfigServerProperties
Note left of A: ConfigurableEnvironment 
Note left of A: TransportConfigCallback
A->B: InitializingBean
Note over B: 生成认证信息CredentialsProvider
B-->>A: repo就解决了用户登陆问题
Note right of B: 上面就是认证
Note right of C: 触发是由于@Bean的Health\n注入的时候触发的
A->C:AbstractScmEnvironmentRepository.findOne\n配置uri,分支等信息
Note right of C: 先后经历了\nConfigServerHealthIndicator\nCompositeEnvironmentRepository\nMultipleEnvironmentRepository\nAbstractScmEnvironmentRepository\nMultipleEnvironmentRepository\nJGitEnvironmentRepository\n触发findOne里面的getLocations方法
C->C:获取配置\n下载配置到本地\nmaster分支
Note right of C: JGitEnvironmentRepository#refresh \n就是我需要的
C-->>A:将运行过程中的配置信息返回 \n 等待其他有需要的请求调用
```







### request请求的实现

这里逻辑很简单，只是在 ResourceController 中定义了接口，然后执行findOne方法，将InputStream信息转成String，显示一下而已。





## 目的

> 看了别人的文章，大概明白了它是这么运行的。
> 所以我希望能够获取下面几个小功能的实现

- [x] 如何漂亮地通过一个注解引入我们要实现的功能
- [x] 配置文件在git的情况兼容处理       
  换句话说，如果我的某些信息是放在github上的，它是怎样处理的。我想将这一部分代码抽出来

- [x] 配置方式支持重试策略 

``` java
@ConditionalOnProperty(value = "spring.cloud.config.fail-fast")
	@ConditionalOnClass({ Retryable.class, Aspect.class, AopAutoConfiguration.class })
	@Configuration
	@EnableRetry(proxyTargetClass = true)
	@Import(AopAutoConfiguration.class)
	@EnableConfigurationProperties(RetryProperties.class)
```

感觉这种方式漂亮得太多了，比我们的系统（我看到过两处）里面的实现更好看更方便

```java
@Retryable(interceptor = "configServerRetryInterceptor")
```

- [ ] 如何将参数注册到Environment上。  那能不能将我们系统里面的配置文件来源都统一在一起处理？appoll ,git,yml,http





## config是如何处理的？







### 另一种引入实现的方式

> 这种方式的好处在于引用可以集中在一起放到application里面，让其他人一目了然。不至于出现上次convenient做了统一精度拦截，结果找不到怎么回事情，最后还是debug发现问题

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(ConfigServerConfiguration.class)
public @interface EnableConfigServer {
	//import bean ConfigServerConfiguration
}
```

```java
/***
* 直接标注了Marker.感觉真的只是起了一个标注的作用
*/
@Configuration
public class ConfigServerConfiguration {
	class Marker {}

	@Bean
	public Marker enableConfigServerMarker() {
		return new Marker();
	}
}
```

```java
@Configuration
@ConditionalOnBean(ConfigServerConfiguration.Marker.class)
@EnableConfigurationProperties(ConfigServerProperties.class)
@Import({ EnvironmentRepositoryConfiguration.class, CompositeConfiguration.class, ResourceRepositoryConfiguration.class,
		ConfigServerEncryptionConfiguration.class, ConfigServerMvcConfiguration.class, TransportConfiguration.class })
public class ConfigServerAutoConfiguration {
		//这里才是真正的重点
  //通过检测到Marker标志符号，自动装配了一个properties & 6 个不同的实现
  //既然这样，我大不了外加一个 @ConditionalOnProperty(value = "com.eggmall.quotation.versionA.enable", matchIfMissing = false) 是不是就能实现在数据库里面改配置，然后走不同的逻辑了？ 这么粗粗地一想，好像还真可以哎。
}
```





### 处理git

其实看了上面关于configServer的实现，原来它也只是用了 [Java org.eclipse.jgit.api.Gi](https://www.kutu66.com/Java_API_Classes/article_69864) 而且我粗粗地看了下代码，发现它好漂亮。

原来已经有这样的实现了啊。





### Retryable

这个好像还只能使用在没有事务的场景吧，换句话说是 http 等外部请求？

```java
		@Retryable(value = {ConnectionException.class}, maxAttempts = 3)
    String push();


    @Recover
    void recover(ConnectionException e);
```

```java
@EnableRetry
@EnableFeignClients
@EnableDiscoveryClient(autoRegister = true)
@SpringBootApplication
public class TestSpringBootApplication 
```





## 参考文章

[解释configServer](https://blog.csdn.net/sinat_25518349/article/details/86323476)

[JGit的sample ，好像很全的样子](https://github.com/centic9/jgit-cookbook)