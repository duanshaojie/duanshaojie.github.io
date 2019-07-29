---
title: 了解gradle的简单使用
tags: gradle
categories: java
---

   在我们的项目开发中，都是使用Gradle进行的项目构建，这里讲讲gradle的一些作用域，和简单的使用。

   <!--more-->

## Gradle如何使用？
   首先我们需要在项目中加入build.gradle文件，这个时候的项目结构如下：
   ```
   spring-demo
        |_src
            |_main
                |_java
                |_resources
            |_test
                |_java
                |_resources
        |_build.gradle
   ```
   gradle默认会从 src/main/java 搜寻打包源码，在 src/test/java 下搜寻测试源码。并且 src/main/resources
   下的所有文件按都会被打包，所有 src/test/resources 下的文件 都会被添加到类路径用以执行测试。所有文件都输出到 build 下，打包的文件输出到 build/libs 下。

##  build.gradle怎么写？
   先贴个项目（多模块构建）中使用的gradle，然后简单解释下用途
   ```
plugins {
    id 'org.springframework.boot' version '2.1.5.RELEASE'
    id 'java'
}

group 'com.edison.demo'
version '1.0-SNAPSHOT'

apply plugin: 'io.spring.dependency-management'
apply plugin: 'java'

//编译环境
sourceCompatibility = 1.8
//运行环境
//targetCompatibility = 1.8
//模块编码
compileJava.options.encoding = 'UTF-8'

//仓库，这里可以自定义远程仓库
repositories {
    mavenCentral()
}
//或者其他仓库
repositories {
    maven {
        url: ''
    }
}

dependencies {
    //依赖模块
    compile project(':demo-business-impl')
    compile("org.springframework.boot:spring-boot-starter-web")
    testCompile("org.springframework.boot:spring-boot-starter-test")
}
   ```

##  依赖

*   compile：运行，发行

*   runtime：该依赖对于运行时是必须的

*   testCompile：仅测试期编译需要

*   providedCompile与providedRuntime：打War包时，不会添加进文件；同compile和runtime

*   implementation：不可以依赖传递

##  发布
这里只介绍简单的push代码，在项目根目录下创建maven_push.gradle
```
apply plugin: 'maven'

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: ""){
            authentication(
                userName: "admin",
                password: "admin")
                }
        }
    }
}
```