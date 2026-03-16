---
layout: post
author: r3tree
title: "SpringCloud笔记"
date: 2021-05-22
music-id: 
permalink: /archives/2021-05-22/1
description: "SpringCloud笔记 Eureka Nacos Feign Docker MinIO Seata"
---

## Eureka注册中心
```java
localhost:8080/user/变成了userservice/user/
给RestTemplate注解@LoadBalanced(userservice/user/请求到 Ribbon负载均衡 找eureka拉取服务)
负载均衡的是orderservice选一个userservice服务里的2个实例
```
## Nacos注册中心
### 分级存储
```java
一级是服务，例如userservice
二级是集群，例如杭州或上海
三级是实例，例如杭州机房的某台部署了userservice的服务器
设置实例的集群属性修改application.yml文件，添加spring.cloud.nacos.discovery.cluster-name属性即可

要做配置的微服务名xxx:ribbon:NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRue 会优先选择本地集群

默认都是临时实例采用心跳监测 非临时实例nacos主动询问 ephemeral: false # 是否是临时实例

YAML配置热更新pattern:dateformat:yyyy/MM/dd
方式一:在@Value注入的变量所在类上添加注解@RefreshScope
方式二@Data@Component@ConfigurationProperties(prefix = "pattern")
public class PatternProperties {private String dateformat;}

多种配置的优先级：[服务名]-[环境].yaml >[服务名].yaml >本地配置
```
## Feign是一个声明式的http客户端(自带ribbon负载均衡) 代替RestTemplate
```java
在order-service的启动类添加注解开启Feign的功能@EnableFeignClients
@FeignClient ("userservice")
public interface UserClient {
    @GetMapping("/user/fid}")
    User findById(@PathVariable("id") Long id);
}

当定义的FeignClient不在SpringBootApplication的扫描包范围时，这些FeignClient无法使用。有两种方式解决：
方式一:指定FeignClient所在包@EnableFeignClients(basePackages = "cn.itcast.feign.clients")
方式二:推荐:指定FeignClient字节码@EnableFeignClients(clients = {UserClient.class})
```
### 服务降级
是服务自我保护的一种方式,或者保护下游服务的一种方式,用于确保服务不会受请求突增影响变得不可用，确保服务不会崩溃
服务降级虽然会导致请求失败，但是不会导致阻塞。  
①：在heima-leadnews-feign-api编写降级逻辑  
②：远程接口中指向降级代码  
③：客户端开启降级  

## Gateway负载均衡相当于在很多orderservice里面选一个
#### Nacos的负载均衡是给orderservice选一个userservice
#### 网关的负载均衡是客户端请求到服务时发生的，而Feign的负载均衡是一个服务调另一个服务时发生的
```java
当过滤器的order值一样时，会按照 defaultFilter > 路由过滤器 > GlobalFilter的顺序执行。
```
## Docker
### 数据卷的作用：将容器与数据分离，解耦合，方便操作容器内数据，保证数据安全
```docker
数据卷操作：
docker volume create
docker volume ls
docker volume inspect查看详情
docker volume rm
docker volume prune删除未使用的
docker run ... -v html:/root/htm：把html数据卷挂载到容器内的/root/html这个目录中

目录挂载与数据卷挂载的语法类似：
-v [宿主机目录]:[容器内目录] 例-v /tmp/mysql/data:/var/lib/mysql
-v [宿主机文件]:[容器内文件] 
```
## MinIO
```dockerfile
docker run -p 9000:9000 --name minio -d --restart=always -e "MINIO_ACCESS_KEY=minio" -e "MINIO_SECRET_KEY=minio123" -v /home/data:/data -v /home/config:/root/.minio minio/minio server /data
```
## Seata
Seata事务管理中有三个重要的角色：  
TC (Transaction Coordinator) - 事务协调者：  
维护全局和分支事务的状态，协调全局事务提交或回滚。  

TM (Transaction Manager)-事务管理器:  
定义`全局事务`的范围、开始全局事务、提交或回滚全局事务。  

RM (Resource Manager)-资源管理器:  
管理`分支事务`处理的资源,与TC交谈以注册分支事务和报告分支事务的状态,并驱动分支事务提交或回滚。  

提供了四种不同的分布式事务解决方案：  
XA模式：强一致性分阶段事务模式，牺牲了一定的可用性，无业务侵入
```java
RM一阶段的工作：
① 注册分支事务到TC
② 执行分支业务sql但不提交
③ 报告执行状态到TC

TC二阶段的工作：
TC检测各分支事务执行状态
a.如果都成功，通知所有RM提交事务
b. 如果有失败,通知所有RM回滚事务

RM二阶段的工作：
接收TC指令，提交或回滚事务

缺点:
因为一阶段需要锁定数据库资源，等待二阶段结束才释放，性能较差
依赖关系型数据库实现事务

1.修改application.yml文件(每个参与事务的微服务),开启XA模式:
seata:
   data-source-proxy-mode: XA # 开启数据源代理的XA模式
2.给发起全局事务的入口方法添加@GlobalTransactional注解，本例中是OrderServicelmpl中的create方法：
@GlobalTransactional
public Long create(Order order) {
   //创建订单
   orderMapper.insert(order);
   //扣余额 ...略 扣减库存 ...略
   return order.getId();
}
```
AT模式弥补了XA模型中资源锁定周期过长的缺陷  
AT模式:最终一致的分阶段事务模式,无业务侵入,也是Seata的`默认模式`
```java
阶段一RM的工作：
注册分支事务
记录undo-log（数据快照）
执行业务sql并提交
报告事务状态

阶段二提交时RM的工作：
删除undo-log即可
阶段二回滚时RM的工作：
根据undo-log恢复数据到更新前

    AT模式的优点：
一阶段完成直接提交事务，释放数据库资源，性能比较好
利用全局锁实现读写隔离
没有代码侵入，框架自动完成回滚和提交
    AT模式的缺点：
两阶段之间属于软状态，属于最终一致
框架的快照功能会影响性能，但比XA模式要好很多

1. 导入课前资料提供的Sql文件: seata-at.sql,其中lock_table表导入到TC服务关联的数据库, undo_log表导入到微服务关联的数据库
2. 修改application.yml文件，将事务模式修改为AT模式即可
        
貌似3.给发起全局事务的入口方法添加@GlobalTransactional注解?
```

TCC模式：最终一致的分阶段事务模式，有业务侵入
```java
TCC模式与AT模式非常相似，每阶段都是独立事务，不同的是TCC通过人工编码来实现数据恢复。需要实现三个方法：
Try：资源的检测和预留；
Confirm：完成资源操作业务；要求 Try 成功 Confirm 一定要能成功。 / Cancel:预留资源释放,可以理解为try的反向操作。

        TCC的优点
一阶段完成直接提交事务，释放数据库资源，性能好
相比AT模型，无需生成快照，无需使用全局锁，性能最强
不依赖数据库事务，而是依赖补偿操作，可以用于非事务型数据库
        TCC的缺点
有代码侵入，需要人为编写try、Confirm和Cancel接口，太麻烦
软状态，事务是最终一致
需要考虑Confirm和Cancel的失败情况，做好幂等处理
        空回滚
当某分支事务的try阶段阻塞时，可能导致全局事务超时而触发二阶段的cancel操作。在未执行try操作时先执行了cancel操作,这时cancel不能做回滚
        业务悬挂
 对于已经空回滚的业务，如果以后继续执行try,就永远不可能confirm或cancel,这就是业务悬挂。应当阻止执行空回滚后的try操作，避免悬挂

@LocalTCC
public interface TCCService{
    //try 这里写 业务悬挂判断
   @TwoPhaseBusinessAction(name = "deduct", commitMethod = "confirm", rollbackMethod = "cancel")
   void deduct(@BusinessActionContextParameter(paramName = "userId") String userId,
               @BusinessActionContextParameter(paramName = "money") int money);
   //确认
   boolean confirm (BusinessActionContext ctx);
   //回滚 这里写 空回滚逻辑 之后 幂等处理 
   boolean cancel (BusinessActionContext ctx);
}

在TCCServiceImpl里加事务
@Transactional
public void deduct(String userId, int money) {
    //这里抛异常回滚 所以加事务
}
```
SAGA模式：SEATA提供的长事务模式，有业务侵入  
```java
一阶段：直接提交本地事务
二阶段：成功则什么都不做;失败则通过编写补偿业务来回滚
        
        Saga模式优点：
事务参与者可以基于事件驱动实现异步调用，吞吐高
 一阶段直接提交事务，无锁，性能好
不用编写TCC中的三个阶段，实现简单
        缺点：
软状态持续时间不确定，时效性差
没有锁，没有事务隔离，会有脏写

```

### Kafka
追求高吞吐量,适合产生大量数据的互联网服务的数据收集业务
### RocketMQ
可靠性要求很高的金融互联网领域,稳定性高,经历了多次阿里双11考验
### RabbitMQ
性能较好,社区活跃度高,数据量没有那么大,优先选择功能比较完备的RabbitMQ
## kafka
```dockerfile
先安装zookeeper
docker run -d --name zookeeper -p 2181:2181 zookeeper:3.4.14

再安装kafka
docker run -d --name kafka \
--env KAFKA_ADVERTISED_HOST_NAME=192.168.200.130 \
--env KAFKA_ZOOKEEPER_CONNECT=192.168.200.130:2181 \
--env KAFKA_ADVERTISED_LISTENERS-PLAINTEXT://192.168.200.130:9092 \
--env KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
--env KAFKA_HEAP_OPTS="-Xmx256M -Xms256M" \
--net=host wurstmeister/kafka:2.12-2.3.1

云主机除外,必须映射 -p 9092:9092
---net=host,直接使用容器宿主机的网络命名空间,即没有独立的网络环境。它使用宿主机的ip和端口
```
### Kafka高可用设计
1. 集群
2. 备份机制(Replication)
   Kafka中消息的备份又叫做副本(Replica)  
   Kafka 定义了两类副本：  
   领导者副本(Leader Replica)  
   追随者副本（Follower Replica)  
   ISR (in-sync replica)需要同步复制保存的follower
   #### 如果leader失效后，需要选出新的leader，选举的原则如下：  
   第一:选举时优先从ISR中选定，因为这个列表中follower的数据是与leader同步的  
   第二:如果ISR列表中的follower都不行了,就只能从其他follower中选取  
   #### 极端情况，就是所有副本都失效了，这时有两种方案  
   第一：等待ISR中的一个活过来，选为Leader，数据可靠，但活过来的时间不确定一  
   第二：选择第一个活过来的Replication，不一定是ISR中的，选为leader，以最快速度恢复可用性，但数据不一定完整 
   
生产者配置
```java
//ack配置 消息确认机制 默认1
prop.put(ProducerConfig.ACKS_CONFIG, "all");
//重试次数10次
prop.put(ProducerConfig.RETRIES_CONFIG, 10);
//数据压缩
prop.put(ProducerConfig.COMPRESSION_TYPE_CONFIG,"snappy");
```
消费者配置  
偏移量提交方式 同步和异步组合提交
## RabbitMQ
### SpringAMQP
提供了模板来发送和接收消息。包含两部分，其中spring-amqp是基础抽象，spring-rabbit是底层的默认实现。
利用RabbitTemplate的convertAndSend方法
```java
消费者 接收消息
@RabbitListener(queues="队列名")

修改application.yml文件,设置preFetch这个值,可以控制预取消息的上限:
preFetch: 1 # 每次只能获取一条消息，处理完成才能获取下一个消息

交换机:FanoutExchange的使用 广播发送
在consumer服务常见一个类，添加@Configuration注解，并声明FanoutExchange、Queue和绑定关系对象Binding
在consumer服务的SpringRabbitListener类中，添加两个方法，分别监听fanout.queue1和fanout.queue2
在publisher服务发送消息到FanoutExchange 以前发送到队列 现在发送到交换机
交换机:Direct Exchange会将接收到的消息根据规则路由到指定的Queue,因此称为路由模式(routes)
可以模拟广播 但是要指定key
@RabbitListener(bindings = @QueueBinding(
value = @Queue(name = "direct.queue1"),
exchange = @Exchange(name = "itcast.direct", type = ExchangeTypes.DIRECT),
key = {"red", "blue"}))
public void listenDirectQueue1(String msg){
System,out.println("消费者1接收到Direct消息：msg
交换机:TopicExchange与DirectExchange类似，区别在于routingKey必须是多个单词的列表，并且以.分割。
Queue与Exchange指定BindingKey时可以使用通配符：
#代指0个或多个单词   *代指一个单词

消息转换器
SpringAMQP中消息的序列化和反序列化是怎么实现的？
利用MessageConverter实现的，默认是JDK的序列化(建议用自定义的MessageConverter)
例
在consumer服务定义MessageConverter:
return new Jackson2JsonMessageConverter();
```
## ES(elasticsearch)
```java
docker run -id --name elasticsearch \
        -p 9200:9200 -p 9300:9300 \
        -v /usr/share/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
        -e "discovery.type=single-node" elasticsearch:7.4.0
        
配置中文分词器ik
把资料中的elasticsearch-analysis-ik-7.4.0.zip上传到服务器上,放到对应目录(plugins)解压

PUT请求 添加映射: http://192.168.200.130:9200/app_info_article
GET请求 查询映射: http://192.168.200.129:9200/app_info_article
DELETE请求 删除索引及映射: http://192.168.200.129:9200/app_info_article
GET请求 查询所有文档: http://192.168.200.129:9200/app_info_article/_search

        
# 创建文档
POST/索引库名/_doc/文档id{json文档}
        
# 查询文档
GET /索引库名/_doc/文档id
        
# 删除文档
DELETE /索引库名/_doc/文档id
        
# 修改文档
全量修改，会删除旧文档，添加新文档
PUT /索引库名/_doc/文档id
{
"字段1": "值1"，
"字段2"："值2"，
//...略
}
增量修改，修改指定字段值(局部修改)
POST /索引库名/_update/文档id
```
### RestClient和RestHighLevelClient
用来操作ES。这些客户端的本质就是组装DSL语句,通过http请求发送给ES。

前者是一个低级客户端,通过Http与elasticsearcha集群进行通信,可以做到负载均衡、故障转移、持久化链接、自动发现集群节点等功能，同时支持所有elasticsearch版本，但是需要自己对请求和相应做编解码(自己写JSON) ;

后者是一个高级客户端,对增删改差进行了封装,不需要处理编解码,类似之前的TransportClient,但是兼容性较差，对客户端和集群版本要求较高。因为RestClient没有提供增删改差方法,只能自己写json并选择Http请求的方法进行实现,一般使用较少,只有RestHighLevelClient无法满足的情况下才会使用。

## MongoDB
```dockerfile
docker run -di --name mongo-service -restart=always \
    -p 27017:27017 \
    -v ~/data/mongodata:/data mongo
```
## xxl-job 分布式任务调度框架
```dockerfile
创建mysql容器，初始化xxl-job的SQL脚本
docker run -p 3306:3306 --name mysql57 \
-v /opt/mysql/conf:/etc/mysql \
-v /opt/mysql/logs:/var/log/mysql \
-v /opt/mysql/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7

创建容器
docker run -e PARAMS="--spring.datasource.url=jdbc:mysql://192.168.200.130:3306/xxl_job?Unicode=true&characterEncoding=UTF-8 \
--spring.datasource.username=root \ 
--spring.datasource.password=root" \
-p 8888:8080 -v /tmp:/data/applogs \
 --name xxl-job-admin --restart=always -d xuxueli/xxl-job-admin:2.3.0
```
分片广播方式：在每一秒中，实例A和B同时接受了10000任务，但根据各自分片id只处理其中的5000个，另外5000个跳过了
## Jenkins 持续集成工具
- Maven Integration plugin： Maven 集成管理插件。
- Docker plugin： Docker集成插件。
- GitLab Plugin： GitLab集成插件。
- Docker Registry私有仓库安装配置

### Jenkins微服务打包配置
```java
clean install -Dmaven.test.skip=true  dockerfile:build -f heima-leadnews/heima-leadnews-service/heima-leadnews-user/pom.xml
```

<font color='red'>注意：根据自己的实际代码路径配置</font>

-Dmaven.test.skip=true  跳过测试

dockerfile:build 启动dockerfile插件构建容器

-f heima-leadnews-user/pom.xml 指定需要构建的文件（必须是pom）

4，并执行shell脚本
```java
if [ -n  "$(docker ps -a -f  name=$JOB_NAME  --format '{{.ID}}' )" ]
 then
 #删除之前的容器
 docker rm -f $(docker ps -a -f  name=$JOB_NAME  --format '{{.ID}}' )
fi
 # 清理镜像
docker image prune -f 
 # 启动docker服务
docker run -d --net=host -e PARAMS="--spring.profiles.active=prod"  --name $JOB_NAME docker_storage/$JOB_NAME
```

## Freemarker指令语法
`gt代替>, FreeMarker会把>解释成FTL标签的结束字符,可使用括号避免这种情况,如:<#if (x>y)>`
```markdown
遍历list
<#list stus as stu> 
    <tr>
        <td>${stu_index + 1}</td> 
        <td>${stu.name}</td> 
        <td>${stu.age}</td> 
        <td>${stu.money}</td>
    </tr>
</#list>

遍历map
<#list stuMap?keys as key>
    <tr>
        <td>${key_index + 1}</td> 
        <td>${stuMap[key].name}</td> 
        <td>${stuMap[key].age}</td> 
        <td>${stuMap[key].money}</td>
    </tr>
</#list>

if指令
<#if stu.name='小红'>...<#else >...
</#if>

空值处理
用法为:variable??,如果该变量存在,返回true,否则返回false
例:为防止stus为空报错可以加上判断如下:
<#if stus??>
<#list stus as stu></#list>
</#if>

缺失变量默认值使用"!"
使用！要以指定一个默认值，当变量为空时显示默认值
例: $fname!'')表示如果name为空显示空字符串。
如果是嵌套对象则建议使用（）括起来
例: $f(stu.name)!''}表示,如果stu或name为空默认显示空字符串。

内建函数语法格式:变量 + ? + 函数名称
1. 集合的大小${集合名?size}
2. 日期格式化
显示年月日： ${today?date}
显示时分秒: ${today?time}
显示日期+时间: ${today?datetime}
自定义格式化: ${today?string("yyyy年MM月")}

3. model.addAttribute("point", 102920122); 输出-> 102,920,122
不想每三位分隔的数字的话${point?c}

4. 将json字符串转成对象text?eval
例子：其中用到了assign标签，assign的作用是定义一个变量。
<#assign text="{'bank':'工商银行','account': '10101920201920212'}" />
<#assign data=text?eval />
开户行：${data.bank} 账号:${data.account}
```


