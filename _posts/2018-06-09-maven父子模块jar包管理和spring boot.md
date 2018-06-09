---
layout: post
title: maven父子模块jar包管理和spring boot
date: 2018-06-09 11:48:00
categories:
    - 工作记录
tags:
    - maven
    - spring boot
    - spring
---

#### 项目结构:
```
parent--父模块空maven项目, 用于管理子模块
    controller
    service
    dao
    model
    client--被其他项目依赖进行微服务内部调用(因下面问题导致client在其他项目中版本冲突和引入大量无用的jar包)
```

#### 存在问题:
1. 父模块继承spring boot和引入大量jar包, 当项目1引入项目2中的client模块时导致项目2中父模块中很多无用的jar包也引入到项目1中
2. 开始计划是父模块放公用的包和管理子模块的版本, 最后每个人的习惯不一样导致不同项目之间的版本冲突
3. spring boot继承在maven父子模块中很容易版本冲突

#### 解决方法:
1. 哪个模块用到什么引入什么, 保持子模块独立
2. 父模块只放\<properties\>标签管理版本, 不引入任何jar包
3. 父模块不继承spring boot, 由controller层导入spring boot

#### 由父模块继承spring boot改为子模块导入spring boot
❌父模块不要这样使用继承
```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.13.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```
✔建议这样使用
```
<!-- 使用dependencyManagement导入版本号 -->
<dependencyManagement>
    <dependencies>
        <!-- 
            自定义版本覆盖spring boot内的版本;
            比如验证框架6.0.10.Final覆盖spring boot parent内的5.3.6.Final;
            项目会优先获取spring boot内版本, 覆盖可以避免版本冲突
            注意: 覆盖版本一定要在spring-boot-dependencies上面
        -->
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-validator</artifactId>
            <version>6.0.10.Final</version>
        </dependency>

        <!-- 导入spring boot的pom -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>1.5.13.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

.....

<!-- 使用导入, 一定要使用spring-boot-maven插件, 否则无法使用java -jar 命令 -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>1.5.13.RELEASE</version>
    <configuration>
        <mainClass>${start-class}</mainClass>
        <layout>ZIP</layout>
        <!--<skip>true</skip>-->
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```