---
layout: post
title: Spring 使用问题（补）
categories: Spring
tags : Spring
author: 彭浩
---

下面为碰到的一些问题
## 问题大全

（1）使用aop时出现，cvc-complex-type.2.4.c: 通配符的匹配很全面, 但无法找到元素 'aop:config' 的声明，但idea已经能够提示  
  该问题可以直接百度搜索，然后复制正确的aop的schema文件即可

（2）报错：Caused by: java.lang.ClassNotFoundException: org.aspectj.weaver.reflect ReflectionWorld$ReflectionWorldException  
该问题正常来说需要导入对应版本的aspectjweaver.jar包，找对应的jar包下载，然后在FILE->Project Structor中引入(+)即可，但是我是采用的gradle构建spring源码阅读环境时并没有加入aspectj的支持和编译配置，所以重新build project又会产生缺少aspect的相关支持的错误，此时参考如下博客进行添加https://www.youyoustudio.com/2019/03/21/109.html，大致步骤是：  
（1）下载aspectj的jar包，直接官网https://www.eclipse.org/aspectj/，  
（2）点击安装，记下安装路径，然后将xxx\lib\aspectj.jar添加到CLASSPATH环境变量中（jdk环境会的配置），以及将xxx\bin添加到PATH环境中，在cmd中使用ajc判断是否安装成功  
（3）在idea中project structer中进行如下步骤

    1. 将Idea的编译器设置为Ajc：
    打开：IDEA--Preferences--Build,Execution,Deployment--Compiler--JavaCompiler,将Use compiler设置为Ajc，将Path to Ajc compiler设置为AspectJ安装目录下的lib文件夹中的aspectjtools.jar文件，同时，可以勾选Delegate to Javac选项，它能够只编译AspectJ的Facets项目，而其他普通项目还是交由Javac来编译。
    2. 将spring-aop_main和spring-aspectjs_main两个模块添加AspectJ Facets：
    打开：File--Project Structure--Facets，点击+号，选择AspectJ，选择spring-aop_main。添加完后，同样的操作，将spring-aspectjs_main模块也设置AspectJ。
    3. 再次执行build
  
  **注意，在我的环境中，无论怎么在创建的项目上导入aspectjweaver.jar，还是会报这个错误，即在external libarires中找到该jar包依然报错，此时采用在Project Structer->platform setting->SDK->classpaths中点击+添加该jar包后该问题解决**

  3、spring执行测试出现问题Error:(26, 38) java: 找不到符号
  符号:   类 InstrumentationSavingAgent
  位置: 程序包 org.springframework.instrument
  问题是模块依赖没有导入只要在Project Structure中为该模块加入模块依赖instrument.main即可

  4、spring整合mybatis时出现Caused by: java.io.IOException: Could not find resource user.mapper.xml 原因是idea不会编译src目录下的xml文件。可行的解决方案是通过

      (1) 我们必须在配置文件中指定mapper.xml文件的位置，例如在springboot项目中，在application.properties中增加：
      mybatis.mapper-locations=classpath:mapper/*.xml
      如果是普通的ssm项目，则这样配置：
```xml
<bean id="sqlSessionFactoryBean" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="druidDataSource"/>
    <property name="configLocation" value="classpath:mybatis-config.xml"/>
    <!-- 配置mapper文件的位置 -->
    <property name="mapperLocations" value="classpath:mapper/*.xml"/>
</bean>
```
    (2)第二种方法：配置maven的pom文件配置，在pom文件中找到<build>节点，添加下列代码：
```xml
<build>  
  <resources>  
    <!-- mapper.xml文件在java目录下 -->
    <resource>  
      <directory>src/main/java</directory>  
        <includes>  
          <include>**/*.xml</include>  
        </includes>  
    </resource>  
    <!-- mapper.xml文件在resources目录下-->
    <--<resource>
        <directory>src/main/resources</directory> 
    </resource>-->
  </resources>  
</build>
```