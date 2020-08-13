---
layout: post
title:  "Maven父子模块统一版本配置方案"
date:   2019-08-13 12:49:08 +0800
categories: 研发效率
tags: Maven
author: 汤波/甘盘
mathjax: true
---

**注意，以下教程请确认MAVEN版本 >= 3.5.0，尤其使用IDEA自带maven的同学务必确认。**
**IDEA 2019.02版本与部分新版本maven(>=3.6.2)不兼容，请更新IDEA到最新版。** 

### 1 修改父Pom的版本。如下：

```
<groupId>com.xxxx.yyyy</groupId>
<artifactId>parent</artifactId>
<version>${revision}</version>
```


### 2 修改主Pom，添加一个插件。如下：
```xml

<build>
   <plugins>
      <plugin>
         <groupId>org.codehaus.mojo</groupId>
         <artifactId>flatten-maven-plugin</artifactId>
         <version>1.1.0</version>
         <configuration>
            <updatePomFile>true</updatePomFile>
            <flattenMode>resolveCiFriendliesOnly</flattenMode>
         </configuration>
         <executions>
            <execution>
               <id>flatten</id>
               <phase>process-resources</phase>
               <goals>
                  <goal>flatten</goal>
               </goals>
            </execution>
            <execution>
               <id>flatten.clean</id>
               <phase>clean</phase>
               <goals>
                  <goal>clean</goal>
               </goals>
            </execution>
         </executions>
      </plugin>
   </plugins>
</build>

```


### 3 修改父Pom中的Properties，新增属性。如下：

```xml

<revision>a.b.c.d-SNAPSHOT</revision>

```

> 其中a,b,c,d一般为数字

### 4 修改子模块的Pom中父版本。如下：

```xml

<parent>
   <groupId>com.juqitech.service</groupId>
   <artifactId>xxxx</artifactId>
   <version>${revision}</version>
</parent>

```



### 5 修改本项目兄弟模块依赖版本号为${project.version}。如下：

```xml

<dependency>
   <groupId>com.xxxx.yyyy</groupId>
   <artifactId>module-01</artifactId>
   <version>${project.version}</version>
</dependency>

```





### 6 参考文献
[1] https://maven.apache.org/maven-ci-friendly.html



* content
{:toc}