---
layout: post
author: r3tree
title: "Java和Mybatis笔记"
date: 2021-05-19
music-id: 
permalink: /archives/2021-05-19/1
description: "Java笔记 Mybatis笔记"
---

# Java笔记
CAS是Compare and Swap（比较并交换）的缩写，也被称为无锁算法。它是一种多线程同步机制，用于解决并发环境下的数据竞争问题。CAS会比较内存位置的当前值与预期值是否相等，如果相等则将内存位置的值更新为新值，否则不做任何操作

幂等性:就是用户对于同一操作发起的一次请求或者多次请求的结果是一致的,不会因为多次点击而产生了副作用。

## 集合框架体系图
![Collection](https://aroucc.oss-cn-hangzhou.aliyuncs.com/images/Collection.png)
## IO流体系
![IO](https://aroucc.oss-cn-hangzhou.aliyuncs.com/images/IO%E6%B5%81%E4%BD%93%E7%B3%BB.png)
## lambda简化
![lambda](https://aroucc.oss-cn-hangzhou.aliyuncs.com/images/%E5%8C%BF%E5%90%8D%E5%86%85%E9%83%A8%E7%B1%BB%E6%88%90lambda.png)
mybatis 1.x和2.x版本区别
![mybatis_2._and_1.](https://aroucc.oss-cn-hangzhou.aliyuncs.com/images/mybatis.png)
```java
null表示对象为空，isEmpty表示值为空

- ArrayList:数组:查询快 增删慢
- LinkedList:双链表:查询慢 增删相对快 首尾元素CURD速度极快 [做队列(排队买票),栈(弹夹)适合]
- HashSet:哈希表:数组(默认长度16加载因子0.75)+链表+(JDK8后链表长度>8,数组长度>=64自动链表转成红黑树)

集合并发修改异常:迭代器遍历使用自己的删除方法.for循环:倒着遍历删除,i--,不能使用增强for循环
```
```java
Java基本类型占用的字节数：
1字节： byte , boolean
2字节： short , char
4字节： int , float
8字节： long , double
编码与中文：
Unicode/GBK： 中文2字节
UTF-8： 中文通常3字节，在拓展B区之后的是4字节
```

```java
StringBuilder线程不安全，StringBuffer线程安全。
对于字符串相关的操作，如频繁的拼接、修改等，建议用StringBuilder，效率更高!
操作字符串较少，或不需要操作，以及定义字符串变量，还是建议用String。
```
### Date和Calendar类的区别：
```java
java.util包提供了这两个类来封装当前的日期和时间。
Date类可以获取日期数据，精确到毫秒;
Calendar类是一个抽象类,它所提供的方法除了可以获取日期数据外,还可以设置和修改日期数据的特定部分,也就是说,Calendar类的功能更强大。
```
### Mybatis参数占位符
```java
mapper里sql语句中的参数占位符
#{...} 执行SQL时，会将#{...}替换为?，生成预编译SQL，会自动设置参数值。 参数传递时使用。
${...}  拼接SQL，直接将参数拼接在SQL语句中，存在SQL注入问题。对表名、列表进行动态设置时使用。
        
模糊查询 where name like '%${name}%' 不能使用#{name} 因为字符串中不能有问号
字符串拼接函数 concat('%',#{name},'%')
```
### Lombok
```java
Lombok 能通过注解的形式自动生成构造器 需要引入依赖
@EqualsAndHashCode 根据类所拥有的非静态字段自动重写 equals方法和 hashCode方法
@Data 提供了更综合的生成代码功能(@Getter + @Setter + @ToString + @EqualsAndHashCode)
@NoArgsConstructor 为实体类生成 无参构造
@AllArgsConstructor 为实体类生成除了static修饰的字段之外 全参构造
@RequiredArgsConstructor 加final的会注入 例 private final IUserServic usrSrvice;
```
### Mybatis主键返回@Options
```java
@Options(keyProperty = "id", useGeneratedKeys = true) 会自动将生成的主键值，赋值给emp对象的id属性
@Insert("insert into emp(... , ...)values(... , ...)
void insert(Emp emp);
```
### Mybatis开启的驼峰camel命名自动映射开关
```java
mybatis.configuration.map-underscore-to-camel-case = true
```
### Mybatis 动态SQL
```xml
<where>会自动去除条件里开头的 and 和 or
    <if test="name != null"> 
        and ...
    </if>
</where>

<set>会删掉额外的逗号（用在update语句中)</set>

<foreach collection="ids遍历的集合名称" item="id遍历出来的元素" 
         separator=",分隔符" open="(遍历开始前拼接的SQL片段" close=")遍历结束后拼接的SQL片段">
    #{id}
</foreach>

<sql id="XXX引用"></sql>先定义sql语句 再调用<include refid="XXX引用"/>
```
### MybatisPlus中比较常用的几个注解如下：
```java
@TableName：用来指定表名
@Tableld:用来指定表中的主键字段信息
@TableField:用来指定表中的普通字段信息
例
@TableName ("tb_user")
public class User {
    @TableId(value= "id", type= IdType.AUTO)AUTO:数据库自增长 不写默认雪花算法
    private Long id;
    @TableField ("username")成员变量名与数据库字段名不一致
    private String name;
    @TableField("is_married")成员变量名以is开头，且是布尔值
    private Boolean isMarried;
    @TableField("order")成员变量名与数据库关键字冲突
    private Integer order;
    @TableField(exist = false)成员变量不是数据库字段
    private String address;
}
```
### 能省去parameterType吗
在MyBatis中，如果你在mapper接口中定义了一个使用List作为参数的方法，并且你希望能够不用显式指定parameterType就能调用这个方法，答案是不能的。

MyBatis需要知道传入的参数是什么类型，以便正确地进行参数映射和SQL执行。即使是传递一个List作为参数，MyBatis也需要知道List中元素的具体类型。如果不指定parameterType，MyBatis将不能正确处理这个方法。

如果你希望避免显式指定parameterType，你可以使用MyBatis的注解来省略这个属性。例如，如果你的方法是传递一个String列表，你可以这样定义：

```java
public interface YourMapper {
@Select("SELECT * FROM your_table WHERE column IN (#{item})")
List<YourType> selectByItems(List<String> items);
}
```
在这个例子中，MyBatis将通过上下文推断出List<String>的parameterType。

如果你正在使用XML配置方式，你需要在`<select>`标签中显式指定parameterType，例如：

```xml
<select id="selectByItems" parameterType="java.util.List" resultType="YourType">
SELECT * FROM your_table WHERE column IN
<foreach item="item" index="index" collection="list" open="(" separator="," close=")">
#{item}
</foreach>
</select>
```
在这个例子中，parameterType被明确指定为java.util.List。

### 使用XML配置方式 能省去parameterType吗

在使用MyBatis的XML配置方式时，通常是不能省略parameterType属性的。MyBatis需要知道传入的参数类型以便正确映射到SQL语句中的占位符。parameterType属性允许你显式地指定传入参数的完整类型。

如果你省略了parameterType属性，MyBatis在运行时可能会遇到类型解析的问题，因为它无法自动推断出传入参数的类型。这可能会导致运行时错误，比如类型不匹配或无法找到合适的映射。

不过，有一种情况下，你可以不使用parameterType属性，那就是当你的参数是一个简单类型（如int、String等）或者MyBatis能够自动推断出参数类型时。例如，如果你有一个只有一个简单类型参数的方法，MyBatis可能能够基于上下文推断出参数类型。

但是，对于复杂类型，特别是像List、Map这样的集合类型，强烈建议显式指定parameterType。这是因为这些类型包含的元素可能具有不同的类型，而MyBatis需要知道这些具体类型才能正确映射到SQL语句中。

总的来说，为了确保MyBatis能够正确解析和执行你的SQL语句，最佳实践是始终在XML映射文件中显式指定parameterType属性，即使对于简单类型的参数也是如此。这有助于避免潜在的类型解析错误和运行时问题。