![](https://b3logfile.com/bing/20180816.jpg?imageView2/1/w/960/h/540/interlace/1/q/100)

## 前言

哈喽大家好，本人最近面试经历有点坎坷，很久没更新了。但我打开公众号发现粉丝居然还涨了，非常感谢各位一直以来的关注，接下来会整理一下最近面试遇到知识点分享给大家。


## Eureka 是什么

Eureka是Netflix开发的服务发现框架，本身是一个基于REST的服务，主要用于定位运行在AWS域中的中间层服务，以达到负载均衡和中间层服务故障转移的目的。SpringCloud将它集成在其子项目spring-cloud-netflix中，以实现SpringCloud的服务发现功能。

## 架构原理

![](https://b3logfile.com/file/2021/06/solo-fetchupload-7072012888315950191-99aef7f7.png)

- Register(服务注册)：把自己的IP和端口注册给Eureka。
- Renew(服务续约)：发送心跳包，每30秒发送一次。告诉Eureka自己还活着。
- Cancel(服务下线)：当provider关闭时会向Eureka发送消息，把自己从服务列表中删除。防止consumer调用到不存在的服务。
- Get Registry(获取服务注册列表)：获取其他服务列表。
- Replicate(集群中数据同步)：eureka集群中的数据复制与同步。
- Make Remote Call(远程调用)：完成服务的远程调用。

Eureka包含两个组件：Eureka Server和Eureka Client。

## Eureka Server

Eureka Server提供服务注册服务，各个节点启动后，会在Eureka Server中进行注册，这样Eureka Server中的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在界面中直观的看到。

　　Eureka Server本身也是一个服务，默认情况下会自动注册到Eureka注册中心。
  
   Eureka Server注册中心**集群中每个节点都是平等的**，集群中的所有节点同时对外提供服务的发现和注册等功能。同时集群中每个Eureka Server节点又是一个微服务，也就是说，每个节点都可以在集群中的其他节点上注册当前服务。又因为每个节点都是注册中心，所以节点之间又可以相互注册当前节点中已注册的服务，并发现其他节点中已注册的服务。

　　如果搭建单机版的Eureka Server注册中心，则需要配置取消Eureka Server的自动注册逻辑。毕竟当前服务注册到当前服务代表的注册中心中是一个说不通的逻辑。



Eureka Server通过Register、Get、Renew等接口提供服务的注册、发现和心跳检测等服务。

Eureka Server为了避免同时读写内存数据结构造成的并发冲突问题，还采用了**多级缓存机制**来进一步提升服务请求的响应速度。

![](https://b3logfile.com/file/2021/06/solo-fetchupload-7893371782018920017-c3f26317.png)

**1、在拉取注册表的时候：**

首先从ReadOnlyCacheMap里查缓存的注册表。
若没有，就找ReadWriteCacheMap里缓存的注册表。
如果还没有，就从内存中获取实际的注册表数据。

**2、在注册表发生变更的时候：**

会在内存中更新变更的注册表数据，同时过期掉ReadWriteCacheMap。
此过程不会影响ReadOnlyCacheMap提供人家查询注册表。

一段时间内（默认30秒），各服务拉取注册表会直接读ReadOnlyCacheMap
30秒过后，Eureka Server的后台线程发现ReadWriteCacheMap已经清空了，也会清空ReadOnlyCacheMap中的缓存
下次有服务拉取注册表，又会从内存中获取最新的数据了，同时填充各个缓存。

**多级缓存机制的优点是什么？**

尽可能保证了内存注册表数据不会出现频繁的读写冲突问题。
并且进一步保证对Eureka Server的大量请求，都是快速从纯内存走，性能极高。


## Eureka Client

Eureka Client是一个java客户端，用于简化与Eureka Server的交互，客户端同时也具备一个内置的、使用轮询(round-robin)负载算法的负载均衡器。

在应用启动后，将会向Eureka Server发送心跳,默认周期为30秒，如果Eureka Server在多个心跳周期内没有接收到某个节点的心跳，Eureka Server将会从服务注册表中把这个服务节点移除(默认90秒)。

Eureka Client分为两个角色，分别是：Application Service(Service Provider)和Application Client(Service Consumer)

通过定时任务30秒拉取一次注册表,30秒发起一次心跳

## 单机版

#### 注册中心依赖

pom.xml

```xml
<!-- spring cloud Eureka Server 启动器 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

#### 注册中心启动类

EurekaServerApplication.java

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

#### 注册中心配置

application.yml

```yml
server:  # 服务端口
  port: 9090  
spring:
  application:  # 应用名字，eureka 会根据它作为服务id
    name: spring-cloud-eureka-server    
eureka:
  instance:
    hostname: localhost
  client:
    service-url:   #  eureka server 的地址， 咱们单实例模式就写自己好了
      defaultZone:  http://localhost:9090/eureka
    register-with-eureka: false  # 不向eureka server 注册自己
    fetch-registry: false  # 不向eureka server 获取服务列表
```

#### 服务提供者依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

#### 服务提供者依赖

```java
@SpringBootApplication
@EnableDiscoveryClient
public class DemoApplication {
	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```

#### 服务提供者配置

application.yml

```yml
server:
  port: 7070
spring:
  application:
    name: spring-cloud-order-service-provider
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:9090/eureka
    fetch-registry: true
    register-with-eureka: true
```

服务提供者与服务消费者配置依赖类似，只是消费者需要restemplate进行消费，而服务提供者需要暴露Controller接口给消费者调用。

## 集群版

1、修改host文件

```
127.0.0.1 EurekaServerA
127.0.0.1 EurekaServerB
```

2、修改application.yml

这里是2个Eureka Server组成的集群，然后u1 的往u2 上面注册，u2往u1上面注册

```yml
spring:
  application:
    name: spring-cloud-eureka-server


---
spring:
  profiles: u1
eureka:
  instance:
    hostname: EurekaServerA
  client:
    service-url:
      defaultZone:  http://EurekaServerB:9091/eureka
    register-with-eureka: true
    fetch-registry: true
server:
  port: 9090
---
spring:
  profiles: u2
eureka:
  instance:
    hostname: EurekaServerB
  client:
    service-url:
      defaultZone:  http://EurekaServerA:9090/eureka
    register-with-eureka: true
    fetch-registry: true
server:
  port: 9091
```

3、修改服务提供者或消费者application.yml

```yml
server:
  port: 7070
spring:
  application:
    name: spring-cloud-order-service-provider
eureka:
  client:
    service-url:
      defaultZone: http://EurekaServerA:9090/eureka,http://EurekaServerB:9091/eureka
    fetch-registry: true
    register-with-eureka: true
  instance:
    prefer-ip-address: true   # 使用ip注册
    #自定义实例显示格式,添加版本号
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}:@project.version@
```

## 服务保护

#### 服务保护模式（自我保护模式）

一般情况下，微服务在Eureka上注册后，会每30秒发送心跳包，Eureka通过心跳来判断服务时候健康，同时会定期删除超过90秒没有发送心跳服务。

　　导致Eureka Server接收不到心跳包的可能：一是微服务自身的原因，二是微服务与Eureka之间的网络故障。通常微服务的自身的故障只会导致个别服务出现故障，一般不会出现大面积故障，而网络故障通常会导致Eureka Server在短时间内无法收到大批心跳。虑到这个区别，Eureka设置了一个阀值，当判断挂掉的服务的数量超过阀值时，Eureka Server认为很大程度上出现了网络故障，将不再删除心跳过期的服务。

　　那么这个阀值是多少呢？Eureka Server在运行期间，会统计心跳失败的比例在15分钟内是否低于85%，如果低于85%，Eureka Server则任务是网络故障，不会删除心跳过期服务。

　　这种服务保护算法叫做Eureka Server的服务保护模式。

　　这种不删除的，90秒没有心跳的服务，称为无效服务，但是还是保存在服务列表中。如果Consumer到注册中心发现服务，则Eureka Server会将所有好的数据（有效服务数据）和坏的数据（无效服务数据）都返回给Consumer。

#### 关闭服务保护模式

```
# 关闭自我保护:true为开启自我保护，false为关闭自我保护
eureka.server.enableSelfPreservation=false
# 清理间隔(单位:毫秒，默认是60*1000)，当服务心跳失效后多久，删除服务。
eureka.server.eviction.interval-timer-in-ms=60000
```

#### 优雅关闭服务

　在Spring Cloud中，可以通过HTTP请求的方式，通知Eureka Client优雅停服，这个请求一旦发送到Eureka Client，那么Eureka Client会发送一个shutdown请求到Eureka Server，Eureka Server接收到这个shutdown请求后，会在服务列表中标记这个服务的状态为down，同时Eureka Client应用自动关闭。这个过程就是优雅停服。

　　如果使用了优雅停服，则不需要再关闭Eureka Server的服务保护模式。

1、POM依赖：

　　优雅停服是通过Eureka Client发起的，所以需要在Eureka Client中增加新的依赖，这个依赖是autuator组件，添加下述依赖即可。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-actuator</artifactId>
</dependency>
```

2、修改全局配置文件：

　　Eureka Client默认不开启优雅停服功能，需要在全局配置文件中新增如下内容：

```
# 启用shutdown，优雅停服功能
endpoints.shutdown.enabled=true
# 禁用密码验证
endpoints.shutdown.sensitive=false
```

3、发起shutdown请求：

必须通过POST请求向Eureka Client发起一个shutdown请求。请求路径为：http://ip:port/shutdown。可以通过任意技术实现，如：HTTPClient、form表单，AJAX等。

　　建议使用优雅停服方式来关闭Application Service/Application Client服务。

## 源码剖析

#### Eureka Server

![](https://gitee.com/fast-zero/img/raw/master/2021-6-13/1623552258094-image.png)

#### Eureka Client

![](https://gitee.com/fast-zero/img/raw/master/2021-6-13/1623559071880-image.png)


## 思考

文章到此结束，那么以下问题是否有答案了呢？

- 1、为什么要用注册中心，用Nginx不行？
- 2、Eureka的client和server是如何工作的，服务注册，服务发现是怎么做到的？
- 3、Eureka集群是如何工作的，一致性能够保证？


## 参考文章

https://blog.csdn.net/zhuyanlin09/article/details/89598245

https://github.com/Netflix/eureka/wiki

https://www.jianshu.com/p/56155d2bde6b

https://cloud.tencent.com/developer/article/1033336

https://www.cnblogs.com/jing99/p/11576133.html

https://www.cnblogs.com/liconglong/p/13223182.html
