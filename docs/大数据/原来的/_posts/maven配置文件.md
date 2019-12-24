---
title: maven配置文件
date: 2019-03-09 11:29:24
author: 陈锦华
password: 123
toc: true
categories: maven
tags:
  - maven
---

## maven配置文件

### 一.配置本地仓库

```
 <!--自定义本地仓库路径-->
<localRepository>E:\JAVA\Maven</localRepository>
```

## 二.国内Maven镜像仓库

```
<mirror>
    <id>alimaven</id>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
<mirror>
    <id>alimaven</id>
    <mirrorOf>central</mirrorOf>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
</mirror>
 
<mirror>
    <id>ibiblio</id>
    <mirrorOf>central</mirrorOf>
    <name>Human Readable Name for this Mirror.</name>
    <url>http://mirrors.ibiblio.org/pub/mirrors/maven2/</url>
</mirror>
<mirror>
    <id>jboss-public-repository-group</id>
    <mirrorOf>central</mirrorOf>
    <name>JBoss Public Repository Group</name>
    <url>http://repository.jboss.org/nexus/content/groups/public</url>
</mirror>
 
<mirror>
    <id>central</id>
    <name>Maven Repository Switchboard</name>
    <url>http://repo1.maven.org/maven2/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
<mirror>
    <id>repo2</id>
    <mirrorOf>central</mirrorOf>
    <name>Human Readable Name for this Mirror.</name>
    <url>http://repo2.maven.org/maven2/</url>
<mirror>
```

## 三.maven项目编译jdk版本更改

首先要下载
maven-compiler-plugin   jar包
通过maven-compiler-plugin  jar包指定JDK版本和编码

方法1
在maven项目的pom.xml中加入一下代码

```
<build>  
    <plugins>  
        <plugin>  
            <groupId>org.apache.maven.plugins</groupId>  
            <artifactId>maven-compiler-plugin</artifactId>  
            <version>2.1</version>  
            <configuration>  
                <source>1.7</source>  
                <target>1.7</target>  
            </configuration>  
        </plugin>  
    </plugins>  
<build> 
```

然后重新update maven就可以解决该问题

方法2
修改maven配置文件

```
    <profile>  
                <id>jdk-1.8</id>  
                <activation>  
                    <activeByDefault>true</activeByDefault>  
                    <jdk>1.8</jdk>  
                </activation>  
                <properties>    
                    <maven.compiler.source>1.8</maven.compiler.source>    
                    <maven.compiler.target>1.8</maven.compiler.target>    
                    <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>    
                </properties>  
        </profile>  
```

