---
layout:     post
title:      "Spring Cloud配置自动更新"
subtitle:   "两种自动更新的配置"
date:       2018-12-22 12:02:00
author:     "yangtao"
header-img: "img/post-bg-2015.jpg"
tags:
    - git 工具
---

> “Spring Boot热更新原理学习”

记录一下Spring Boot配置自动更新的原理
## 配置中心介绍
顾名思义，配置中心就是提供一个界面供用户统一管理配置，实现配置的热更新。
Spring Cloud配置中心大概有以下几种选型：
- Spring cloud config——存储在git中，变更配置需要动态刷新，调refresh
- DisConf——百度开源，基于Zookeeper
- Apollo——携程，Java .Net，客户端和配置中心通过HTTP长链接实现实时推送

公司项目使用腾讯TSF，是一套基于Spring Cloud的微服务框架，配置采用的是Consul。暂时没搞明白consul做配置中心的实现，待学习。

## Spring boot配置的使用

Spring boot代码里引入配置，主要有@ConfigurationProperties与@Value两种方式

一种是使用@ConfigurationProperties注解来定义配置类，配置类里的成员变量名与配置里的key对应，成员变量值与value对应。进一步的，配置类里还可以定义map与list成员来接收嵌套的配置。

另一种是使用@Value注解,在需要使用配置的class中按下面格式导入即可
```
    @Value("${refreshTest.value}")
    private String value;
```

其中@ConfigurationProperties会热更新，而@Value引入的注解需要在类上添加@RereshScope注解
## 热更新原理
### ContextRefresher
Spring Cloud的配置更新基于此类，在收到配置中心的更新之后（原理估计与调refresh接口类似），调用ContextRefresher类的refresh方法。refresh方法做了两件事
1. extract 接收当前环境所有属性源 PropertySource,将非标准属性源的属性汇总到一个map返回,标准属性源即不需要更新的属性源（系统属性等）
2. addConfigFilesToEnvironment，核心逻辑，创建一个新的Spring boot应用并初始化，读入新的配置。
    1. 将有更新的key收集起来，发送EnvironmentChangeEvent事件。
    2. 调用 RefreshScope.refreshAll 方法。

#### EnvironmentChangeEvent事件
Spring Cloud收到EnvironmentChangeEvent事件会重新绑定使用了@ConfigurationProperties 注解的 Spring Bean。

#### @RefreshScope
使用了@RefreshScope这个注解的bean收到事件会自动更新。@RefreshScope里这个Scope与我们使用IOC的protoType与singleton是一类东西，定义了bean的生命周期与使用方式等。

所有 @RefreshScope 的 Bean 都是延迟加载的，只有在第一次访问时才会初始化。这个类中有一个成员变量cache，用于缓存所有已经生成的Bean，在调用get方法时尝试从缓存加载，实现自动更新只需要清空缓存即可。

## 后记
本文基于 [Spring Cloud 是如何实现热更新的](http://www.scienjus.com/spring-cloud-refresh/),Spring的源码还需要进一步学习