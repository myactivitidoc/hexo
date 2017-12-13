---
layout: post
title: myActiviti 是什么
description: 工作流是一个能够提高企业级应用开发生产力工具，而 myActiviti 是我们对它的扩展。
category: 工作流
date: 2017-12-06
list_number: false
---

## [为什么要开发 flying](#为什么要开发-flying)
众所周知，Mybatis 在设计之初是作为一个工作于单线程之下的中间件而存在，虽然它易于上手，但放到互联网环境下使用时，初学者不可避免的要面对诸如“一级缓存存在脏数据”、“需要写大量明文 SQL 语句”等问题。这些是用户最关心的“是否可用”级别问题，也是 Mybatis-3 系列版本中开发团队并没有完全解决的问题，或者说，相对于另一个竞争产品 Hibernate，Mybatis 的开发团队选择了一种更谦逊的方式，他们开放 Mybatis 接口，允许用户开发插件，按自己的方式来解决这些问题。于是，一切 ORM 领域相关的问题在 Mybatis 上通过插件都有了解决方案。而本文所要介绍的 flying，则是集成了用户最需要的几个功能的插件组。与此同时 flying 提出了一种新的调用数据的方式，希望能对您起到抛砖引玉的作用。

## [如何获取 flying](#如何获取-flying)

目前 flying 已加入到 maven 中心库内，所以您只需在POM文件中加入依赖：
```xml
<groupId>com.github.limeng32</groupId>
<artifactId>mybatis.flying</artifactId>
<version>0.9.2</version>
```
即可以使用。mybatis 版本与 flying 最新版本的对应关系见下：

| mybatis 版本     | flying 版本   |
|:--------|:-------:|
| `3.2.6` `3.2.7` `3.2.8` | `0.7.4` |
| `3.3.0` `3.3.1` | `0.8.2` |
| `3.4.0` `3.4.1` `3.4.2` `3.4.3` `3.4.4` `3.4.5` | `0.9.2` |
（目前 flying 不支持 mybatis-3.2.6 之前的版本）

考虑到您可能是以前版本 flying 的使用者，我们会在文档中用 `最新版本新增` 将最新版本新增特性标识出来，在目录中用<font color="red">红色</font>将新增或有修改的章节标识出来。

如果您使用的是基于某一版本 mybatis 的定制版，您可以在依赖配置中加入 `exclusions`，以 mybatis-3.3.x 的定制版为例，您的依赖配置就像下面这样：
```xml
<dependency>
    <groupId>com.github.limeng32</groupId>
    <artifactId>mybatis.flying</artifactId>
    <version>0.8.1</version>
    <exclusions>
        <exclusion>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```
在您做好以上配置之后，flying 会自动下载到您的电脑本地。

## [如何使用 flying](#如何使用-flying)

使用本插件前您需要具备 Java 开发数据库应用的知识和使用 Mybatis 基本功能的经验。
在 Mybatis 中，有一个核心的配置文件，称做 Configuration.xml，我们的插件需要先在这里配置，才能在运行期被识别出来。

在 Configuration.xml 中，首先为了让 Mybatis 在多线程环境下工作的更好，需要在 `settings` 中进行以下设置：
```xml
<settings>
    <setting name="lazyLoadingEnabled" value="false" />
    <setting name="aggressiveLazyLoading" value="false" />
    <setting name="localCacheScope" value="SESSION" />
    <setting name="autoMappingBehavior" value="PARTIAL" />
    <setting name="lazyLoadTriggerMethods" value="" />
</settings>
```
（以上配置中没有提到的内容用户可按照自己的实际情况进行配置）
 
然后，在 Configuration.xml 的底部，我们加入 `plugins` 元素，将 flying 中的 Interceptor（拦截器）配置成 `plugin`，然后放到 `plugins` 中。目前 flying 有两个主要拦截器（负责处理 pojo 自动注射的 `AutoMapperInterceptor` 和负责完善二级缓存的 `EnhancedCachingInterceptor`），配置它们的方式如下：
```xml
<plugins>
    <plugin interceptor="indi.mybatis.flying.interceptors.AutoMapperInterceptor">
        <property name="dialect" value="mysql" />
    </plugin>
    <plugin interceptor="indi.mybatis.flying.interceptors.EnhancedCachingInterceptor">
        <property name="cacheEnabled" value="true" />
    </plugin>
</plugins>
```
（以上配置中 `dialect` 标识了目标数据库为 mysql，`cacheEnabled` 标识了二级缓存为打开状态，更多的属性请参见讲解该插件的章节。）

配置好以上内容之后，保存 Configuration.xml 文件，按正常方式启动 Mybatis，flying 即可生效。

flying采用非侵占式工作机制，您既可以在以传统方式使用 mybatis 的项目中加入一部分以 flying 方式配置的方法，也可以在以 flying 方式配置的项目中加入一部分传统方式的方法，当然更好的选择是，完全使用 flying 方式替代传统的方式。
 
## [如何查看 flying 的源代码](#如何查看-flying-的源代码)
 
flying 的代码在 github 和 gitee 上进行开源，前者访问地址为 [https://github.com/limeng32/mybatis.flying](https://github.com/limeng32/mybatis.flying)，后者访问地址为[https://gitee.com/limeng32/mybatis.flying](https://gitee.com/limeng32/mybatis.flying) 您可以通过 Issues 与我们联系，提出您的需要，或是加入我们成为贡献者，提交您的代码，我们非常欢迎您这样做。