# 微服务开发为什么要用阿里巴巴的 `Nacos`

> 本文适合有 Java 基础知识的人群

![](./images/0.png)

<p align="center">本文作者：HelloGitHub-<strong>秦人</strong></p>

HelloGitHub 推出的[《讲解开源项目》](https://github.com/HelloGitHub-Team/Article)系列，今天给大家带来一款开源 Java 版可以实现动态服务发现，配置和服务管理平台——nacos，使用过 `Spring Cloud` 的的伙伴应该用户服务注册中心 `Eureka`，今天的`Nacos`的功能比 `Eureka` 更强大。

> 项目源码地址：https://github.com/alibaba/nacos

## 一、项目介绍
Nacos 提供了一组简单易用的特性集，帮助开发者快速实现动态服务发现、服务配置、服务元数据及流量管理。
`Nacos`的主要特性：
- 服务发现：Nacos 支持基于 DNS 和基于 RPC 的服务发现。服务提供者使用 原生SDK、OpenAPI、或一个独立的Agent TODO注册 Service 后，服务消费者可以使用DNS TODO 或HTTP&API查找和发现服务。
- 服务健康监测：Nacos 提供对服务的实时的健康检查，阻止向不健康的主机或服务实例发送请求。
- 动态配置服务：动态配置服务可以让您以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置。
- 动态 DNS 服务：动态 DNS 服务支持权重路由，使用者更容易地实现中间层负载均衡、更灵活的路由策略、流量控制以及数据中心内网的简单DNS解析服务。
- 服务及其元数据管理：Nacos 能让使用者从微服务平台建设的视角管理数据中心的所有服务及元数据，包括管理服务的描述、生命周期、服务的静态依赖分析、服务的健康状态、服务的流量管理、路由及安全策略、服务的 SLA 以及最首要的 metrics 统计数据。

`Nacos`生态图

![](./images/1.png)

## 二、`SpringBoot` 实战

`Nacos` 主要的功能有注册中心和配置中心。下面主要介绍这两块功能的使用。章节2.2是官方教程，有官方源码可直接下载；章节2.3是实战演练，创建两个微服务：提供者，消费者的形式使用`nacos`的这两大功能。各位伙伴可快速切换阅读。

### 2.1运行`nacos`

下载地址：https://github.com/alibaba/nacos/releases
```bash
unzip nacos-server-$version.zip  #解压
cd nacos/bin
startup.cmd -m standalone #单机模式
```

**访问首页**</br>
nacos的访问地址：http://localhost:8848/nacos/
默认账号密码：nacos nacos
页面截图如下：
![](./images/2.png)

### 2.2实战

#### 2.2.1 配置中心

市面上存在的的微服务配置中心有consul,config,而Nacos作为配置中心的优势就是支持热部署。

**创建微服务项目**
创建`SpringBoot`项目主要有三种方式：通过网站创建，`IntelliJ IDEA`的`Spring Initializr`工具创建，Maven 创建项目形式创建。项目的`pom` 文件内容如下：
```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!--nacos-config的Spring cloud依赖  -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        <version>0.9.0.RELEASE</version>
    </dependency>
```
**bootstrap.yml配置**
```yml
spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        prefix: nacos-config
  profiles:
    active: dev
```
**Nacos配置**</br>
`Nacos`上创建配置文件名称格式：**${prefix}-${spring.profile.active}.${file-extension}**，如上一步`bootstrap.yml`的配置可知，我要创建的配置名为：`nacos-config-dev.yaml`,内容如下：
![](./images/3.png)

**创建Controller**
动态获取用户名称的功能为例，代码如下：
```java
@RestController
@RefreshScope
public class ConfigController {

    @Value("${username:wangzg}")
    private String username;

    @RequestMapping("/username")
    public String userNameInfo() {
        return username;
    }
}
```
注意：`Controller`上要添加`@RefreshScope注解`，它实现了配置的热加载。

**验证结果**
本地运行项目，可以看到项目的启动时，端口已变为我们在`Nacos`上配置的端口`8090`。
![](./images/4.png)

在浏览器访问链接：`http://localhost:8090/username`,返回`testuser`。修改`Nacos`上`username`的值，不需要重启微服务，重新请求链接，`username`的值会动态变。可见nacos作为配置中心实现了热加载功能。

#### 2.2.2 注册中心

 1. 创建服务提供者</br>
创建微服务可参上上一步`配置中心`的创建方式，新建一个`Controller`，创建一个对提供的接口，代码如下：
```java
@RestController
public class ProviderController {

    @GetMapping("/sayHello")
    public String sayHello(@RequestParam(value = "name",defaultValue = "helloWord")String sayHello){

        return "tom say: " + sayHello;
    }
}
```
启动服务，访问地址：http://localhost:8099/sayHello,可输出：
`tom say: helloWord`,表示微服务以创建成功。

 2. 创建服务消费者</br>
这里采用`FeignClient`的方式完成跨服务间调用（有兴趣的同学也可以研究一下RestTemplate的方式）。 

**pom文件**</br>
在nacos-consumer的pom文件要添加`Feigin-Client`的maven依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
**添加注解**</br>
在微服务启动类`*Application.java`添加注解`@EnableFeignClients`。

**创建FeignClient**
```java
@FeignClient("nacos-provider")
public interface ProviderClient {

    @GetMapping("/sayHello")
    String sayHello(@RequestParam(value = "name", defaultValue = "wangzg", required = false) String name);
}
```
说明：FeignClient注解传入的`name`,指定FeignClient的名称，如果项目使用了Ribbon，name属性会作为微服务的名称，用于服务发现。

**创建ConsumerController**<br/>
```java
@RestController
public class ConsumerController {

    @Autowired
    ProviderClient providerClient;

    @GetMapping("/hi-feign")
    public String hiFeign(){
       return providerClient.sayHello("feign");
    }
}
```
重启工程，在浏览器上访问http://localhost:8090/hi-feign，可以在浏览器上展示正确的响应，这时nacos-consumer调用nacos-provider服务成功。


下面一张请求流转的时序图，这样理解清晰一些。   
![](./images/5.png)

项目地址：

## 三、最后

本篇文章通过服务提供者，服务消费者给大家讲解了`Nacos`的服务注册发现功能；用动态获取用户名的例子让大家感受一下`Nacos`的动态配置功能。可以也有一些讲的不对的对方，欢迎大家批评指正。

服务注册与发现，动态配置这两大块功能在微服务开发中特别重要，而阿里巴巴的`Nacos` 集成了注册中心和配置中心的功功能，难道不香吗？

教程至此，你应该也能对 `Nacos` 有一些感觉了吧。新工具可能会带来飞一般的感觉，参考我上面的案例，实践一下，在实践中会发现很多乐趣！

## 四、参考资料
- [官方文档](https://nacos.io/zh-cn/docs/what-is-nacos.html): https://nacos.io/zh-cn/docs/what-is-nacos.html
- [Nacos作为配置中心](https://blog.csdn.net/forezp/article/details/90729945): https://blog.csdn.net/forezp/article/details/90729945