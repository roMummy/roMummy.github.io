---
title: 连接mysql
date: 2018-07-12 17:47:09
tags: 随记
categories: SpringBoot
---
最近在学习springboot，刚刚学到了连接数据库这里。作为小白的我遇见一个奇怪的错误
```
### Cause: org.springframework.jdbc.CannotGetJdbcConnectionException: Failed to obtain JDBC Connection; nested exception is com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
```
配置信息
```
spring.datasource.url=jdbc:mysql://localhost:3306/test1?useSSL=true
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

>原来是开启了ssl加密连接方式。但是又没有配置ssl加密，所以导致不能连接上数据库。
mysql高版本都推荐使用ssl的方式连接数据库。
[配置方法](https://dev.mysql.com/doc/refman/5.7/en/using-encrypted-connections.html#using-encrypted-connections-server-side-configuration)

>不过不需要使用ssl这种连接方式的话可以使用useSSL=false避免这个错误 ，或者使用verifyServerCertificate=false不使用证书配置
```
spring.datasource.url=jdbc:mysql://localhost:3306/test1?useSSL=true&verifyServerCertificate=false
```