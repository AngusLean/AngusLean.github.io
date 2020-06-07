---
title: maven项目实践
date: 2018/1/26
updated: 2018/3/14
tags:
categories:
   - Programming Language
   - Java
   - 构建工具
---
作为Java世界事实上的构建标准，maven被几乎所有的项目使用着。 但是如果任何工具一样，总会有或多或少的最佳实践或者说避坑选项。 本节尝试列出根据我的经验所“认为”的一些实践经验， 不敢称为“最佳实践”，“项目实践”姑且可以。

本文假设具备基础的maven使用经验，了解如何使用maven,如何添加依赖如何打包。 本文的目的仅仅是介绍maven在项目中一些常用、非常重要的概念， 使得开发人员对于maven有个整体的概念。 具体的配置需要另行查阅文档。

<!--more-->


### 模块及版本管理
#### 包定义
maven采用3个属性定义一个包：
- goupId
- artifactId
- version

这几个可以定义一个具体的包，他们通常都是小写。 `groupId`定义了你的组织名称，比如`org.springframework` ,`artifactId`则是包(模块)名称，比如`spring`。`version`是版本。其中，`version`分为如下两种：

- 快照版（**以`-SNAPSHOT`结尾**）
- 稳定版（ ** 没有以`-SNAPSHOT`结尾**）

两者的区别是  快照版 的在每次使用maven构建的时候，都会去*maven仓库*查询是否有新版本。如果有则会更新本地缓存。 而稳定版则会优先使用本地缓存的包，仅在本地缓存没有的情况下才会去*maven仓库*寻找并下载。

#### 仓库
*maven仓库*这里有2个概念：
- *maven仓库*
默认情况下，这个指的是`search.maven.org`。 项目中推荐自己搭建一个maven仓库，使得项目源码可以方便的推送上去而不用担心安全问题，通常称为 私库， 使用`nexus`
- 本地缓存
默认情况下，是用户目录下的`.m2`目录，这里就相当于上面的*maven仓库*在本地的一个缓存，构建时下载过的包都会在这里存在一份。 对于非快照版的依赖，则会优先使用本地缓存的包。

所以一个建议就是对于一个多模块项目， 开发时模块之间的互相依赖使用快照版本，上线时使用稳定版本。*对于外部依赖项，任何时候都建议使用稳定版而不是开发版*

#### `pom.xml`配置建议
对于定义版本信息，非常建议通过属性`properties`的方式来声明，便于管理。例如可以声明一个sping版本的`properties`，所有依赖的spring各种包都使用这个属性来指定版本，便于统一管理。

每个模块都有其模块打包目标，比如是一个web项目，通常就会是`war`包；对于一个父模块，会是`pom`；对于一个可执行项目，会是`jar`包（默认就是）。 这个通过在对应的`pom.xml`中配置`<packaging>pom</packaging>`

### 依赖管理
依赖管理是一个构建工具最为重要的功能，C系使用`makefile`来描述一个项目的依赖项，但是显得有点晦涩难懂。maven的则要简单的多，通过在POM文件中添加指定的包信息即可，maven会自动的在网络或者本地缓存中搜寻指定的包。 添加一个依赖时注意如下的事项：

- 依赖传递
比如，你依赖一个A包，A包依赖B包，B包依赖C包。 那么你在添加A包依赖的时候同时就添加了B，C 2个包的依赖，没有特殊处理的情况下，在打包的时候，他们都会被打进最终的包中。
有一个特殊配置项`exclusions`，可以用来指定在这个依赖中被排除的包。 比如上面的例子，如果在`exclusions`中添加了C的项，那么最终打的包就不会包含C这个包。对于一些较大的依赖项（典型的就是dubbo），是非常有必要的。 否则，项目中很可能会出现很多同一个库不同的版本，这样很容易引起API之间的兼容性问题。

- 依赖作用范围
上面说到的依赖项还有一个配置项，就是`scope`。maven的`scope`有6种值：

  - `compile`
默认的范围就是他。 在所有的`ClassPath`中都可见
  - `provided`
典型使用场景是`servlet api`，我们希望使用web容器（比如tomcat）提供的时候就可以选择这个范围。
  - `runtime`
编译时不需要，运行时才需要。比如JDBC的实现，编译的时候只需要用到JDK的JDBC API，但是运行时肯定需要具体实现（这里涉及到一个叫做SPI的机制）
  - `test`
仅在测试中需要，`CLASSPATH`也仅限于`test`
  - `system`
类似`provided`，但是需要显示的提供jar
  - `import`
仅在`dependencyManagement`中的`type`为`POM`时有效，仅仅指定依赖的位置。 这个标签下文会解释。

在了解上面的2项过后，添加依赖时就可以做到大概心中有数了。 但是对于一个模块较多的项目，比如如下的结构：

--A
----A1
-------A1-1
----A2
-------A2-1
-------A2-2

该项目有A和B两个模块， A中又分为A1，A2两个模块，其中，A1还有子模块，等等。这种情况，如果A中定义了某些依赖，A1中又定义一些，A1-1又定义一些，稍有不同，他们定义的还是不同的版本，在管理起来就会非常麻烦。maven提供了一种依赖管理的机制，也就是`dependencyManagement`
`dependencyManagement`是一种集中式的依赖管理，也就是说，可以把一个模块的所有依赖都声明在一个`dependencyManagement`中（特别是定义版本信息），子模块继承过去再声明需要哪些包以及对应的类型等信息，但是不用再指定版本信息。 使用这种方式可以规避多模块项目中由于子模块添加了不同版本依赖导致的兼容性问题。
使用方式也很简单，直接在根POM中使用`dependencyManagement`包围`dependencies`，其中的每个`dependency`指明版本及`exclusions`等信息，子模块仅需声明`goupId,artifactId`。

### 构建管理
maven构建分为多个阶段。默认情况下，有如下几个阶段：
- `validate`
- `compile`
- `test`
- `package`
打包当前项目，比如web项目为war包（需要手动指定），普通java项目为jar包
- `verify`
- `install`
安装到本地缓存，前面的模块及版本管理说过
- `deploy`
发布到maven仓库中（一般会是自己搭建的maven私库）

调用`mvn install`会顺序调用`validate, compile, test, package，verify，install`。而`install`命令会将所有的包安装到本地的`.m2`目录，如同前面所说，其他模块依赖了当前`install`的包的话则会使用本地的版本匹配的包，而快照的依赖则会去配置的仓库中寻找、并且和本地的对比谁是最新的然后使用它。

调用`mvn deploy`会自动调用构建等过程，然后将指定包推送到`POM`文件中配置的`distributionManagement`节点地址中去。 如果其他项目引用了当前项目的包，推送上去过后， 其他项目打包时则会根据上面说的快照与非快照依赖更新依赖包。


### 插件管理
插件是maven依赖生存的工具，上面构建的几个阶段其实就是通过相应的插件来完成的。 默认的几个阶段对于单纯打包来说够用，但是通常一个项目还需要配置其他项。 如下是几个非常典型的

#### 编译、字节码的JDK配置
通常会使用 `maven-compiler-plugin` 插件来指定编译当前项目使用的JDK版本以及生成的字节码的版本。

#### 资源文件编码
通常会使用 `maven-resources-plugin` 来配置编码为`utf-8`.
同时，该插件还常常用来包含需要包含在最终包中的额外资源，比如包含某个`lib`,`word、excel`等等。

#### 打包管理
通常会使用`maven-assembly-plugin`插件来将源码打包成某种结构的压缩包，以便于编写shell脚本执行项目。 一个通常的配置:

```xml
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <descriptor>src/main/assembly/assembly.xml</descriptor>
        <finalName>${project.artifactId}</finalName>
    </configuration>
    <executions>
        <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
其中`assembly.xml`如下

```xml
<assembly>
    <id>assembly</id>
    <formats>
        <format>tar.gz</format>
    </formats>
    <includeBaseDirectory>true</includeBaseDirectory>
    <fileSets>
        <fileSet>
            <directory>src/main/assembly/bin</directory>
            <outputDirectory>bin</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
        <fileSet>
            <directory>src/main/assembly/conf</directory>
            <outputDirectory>conf</outputDirectory>
            <fileMode>0644</fileMode>
        </fileSet>
    </fileSets>
    <dependencySets>
        <dependencySet>
            <outputDirectory>lib</outputDirectory>
        </dependencySet>
    </dependencySets>
</assembly>
```