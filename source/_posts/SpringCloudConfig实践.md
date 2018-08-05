---
title: SpringCloudConfig实践
date: 2018-08-05 17:54:12
tags:
- SpringCloud
- SpringCloudConfig
categories:
- SpringCloud

---

<i>一些使用SpringCloudConfig的小技巧</i>

<!-- more -->

## 为啥要用Spring Cloud Config
Spring  Cloud Config可以提供配置文件的区分profile，区分application，区分version管理，减少部署复杂度。这里只考虑以git仓库作为配置文件载体的情况。

## 实践中遇到的问题
先引用一段文档

>The server is a Spring Boot application so you can run it from your IDE instead if you prefer (the main class is ConfigServerApplication). Then try out a client:


```
$ curl localhost:8888/foo/development

{"name":"development","label":"master","propertySources":[
  {"name":"https://github.com/scratches/config-repo/foo-development.properties","source":{"bar":"spam"}},
  {"name":"https://github.com/scratches/config-repo/foo.properties","source":{"foo":"bar"}}

]}
```

>The default strategy for locating property sources is to clone a git repository (at spring.cloud.config.server.git.uri) and use it to initialize a mini SpringApplication. The mini-application’s Environment is used to enumerate property sources and publish them via a JSON endpoint.

```
The HTTP service has resources in the form:

/{application}/{profile}[/{label}]

/{application}-{profile}.yml

/{label}/{application}-{profile}.yml

/{application}-{profile}.properties

/{label}/{application}-{profile}.properties

```

上面文档的大致说明了，通过ConfigServer获取配置的Http请求方式，但并没有说明在存储的yml文件的命名方式和存储方式。事实上，在仓库中存储的文件命名可以遵循{application}-{profile}.yml的命名方式，就跟在resourses文件夹里的一样。上面http请求中的label参数可以是git branch,git tag，那种都可以，也就是区分版本实际上使用的是git进行版本控制。

#### 问题就来了：

有两个application,clientA与clientB，想要两个application共享同一个config.yml文件

config.yml

```
user.name: ln

```
## 共享yml文件
在共享同一yml文件之前还有一个问题要先解决，就是如何加载多个yml配置文件。

可以通过spring.profiles.include 来完成。

例如：

在git仓库中有

```
|- clientA-local.yml
|- clientA-user.yml
```

clientA-local.yml

```
spring.profiles.include: ["user"]
```
clientA-user.yml

```
user.name: ln
```
按照以上配置时，再请求 http://configServer/clientA/local/。

就可以发现clientA-user.yml中的配置已经加载进来了。具体的原理就是通过建立多个profile再通过spring.profiles.include整合配置。

回到共享config的问题。

我在翻阅了SpringCloudConfigServer的源码之后发现了神奇的东西。在org.springframework.cloud.config.server.environment.NativeEnvironmentRepository中有以下代码：

```
@ConfigurationProperties("spring.cloud.config.server.native")public class NativeEnvironmentRepository implements EnvironmentRepository, SearchPathLocator, Ordered {

private String[] getArgs(String application, String profile, String label) {
    List list = new ArrayList();
    String config = application;
    if (!config.startsWith("application")) {
        config = "application," + config;
    }    
    list.add("--spring.config.name=" + config);    
    list.add("--spring.cloud.bootstrap.enabled=false");
    list.add("--encrypt.failOnError=" + this.failOnError);
    list.add("--spring.config.location=" +            StringUtils.arrayToCommaDelimitedString(getLocations(application, profile,         label).getLocations()));
    return list.toArray(new String[0]);}

}
```

NativeEnvironmentRepository是用于查询和筛选配置文件的类，其中最重要的方法是findOne,findOne中调用getArgs来获取查询和筛选参数。其中最重要的是这一句：

```
 if (!config.startsWith("application")) {     
    config = "application," + config;
 }    
```
如果传递的config没有以application开头的话，那就把application加进去。最终的结果便是application-{profile}.yml中的配置可以被{application}-{another-profile}.yml中通过spring.profiles.include=[{profile}]加载进去。

例如：

在git仓库中有

```
|- clientA-local.yml
|- clientA-user.yml
```

clientA-local.yml

```
spring.profiles.include: ["user"]
application-user.yml
user.name: ln
```

按照以上配置时，再请求 http://configServer/clientA/local/。

可以发现application-user.yml中的配置已经加载进来了。

## 仓库整理的问题
上面基本解决了很多问题，因为在实际只用时，同一个环境的数据库配置，各种签名秘钥的配置都是相同的，减少了重复写配置的问题。但时间一长又出现了另一个问题，就是仓库里太乱了。

例如：

```
|- clientA-local.yml
|- clientA-prod.yml
|- clientB-local.yml
|- clientB-prod.yml
|- application-user.yml

```
理想中的形式应该是这样的：

```
|- clientA
    |- clientA-local.yml
    |- clientA-prod.yml

|- clientB
    |- clientB-local.yml
    |- clientB-prod.yml
|- application-user.yml

```

但是如果直接把形式改掉会发现找不到配置。解决方式如下：

在configServer的配置中加入：

```
spring.cloud.config.server.git.search-paths: '{application}/{profile},{application}'
```

这样的话，请求 http://configServer/clientA/local/ 会搜索 / 目录、/clientA 目录、 /clientA/local 目录下的配置，扩大了配置检索的范围。

解决了这个问题，SpringCloudConfig使用起来就比较开心了。

## Demo
Application

https://github.com/guoyxln/SpringCloudConfigDemo

配置仓库

https://github.com/guoyxln/SpringCloudConfigDemoRepository

使用的时候把config/src/resources/application.yml中的git username 和 password 换成自己的github账号密码就好了。

config-client 提供了个简单的name接口用于显示从配置中读取的name。可以试一下。

