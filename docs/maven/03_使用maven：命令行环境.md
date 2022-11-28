

# 使用Maven：命令行环境



## 1.根据坐标创建maven工程



### 1.1 maven核心概念：坐标

#### 1.1.1 数学中的坐标

![image-20221022204123161](03_使用maven：命令行环境/image-20221022204123161.png)

使用 x、y、z 三个**『向量』**作为空间的坐标系，可以在**『空间』**中唯一的定位到一个**『点』**。

#### 1.1.2 Maven中的坐标

使用三个**『向量』**在**『Maven的仓库』**中**唯一**的定位到一个**『jar』**包。

- **groupId**：公司或组织的 id
- **artifactId**：一个项目或者是项目中的一个模块的 id
- **version**：版本号


三个向量的取值方式

- groupId：公司或组织域名的倒序，通常也会加上项目名称
  - 例如：com.atguigu.maven
- artifactId：模块的名称，将来作为 Maven 工程的工程名
- version：模块的版本号，根据自己的需要设定
  - 例如：SNAPSHOT 表示快照版本，正在迭代过程中，不稳定的版本
  - 例如：RELEASE 表示正式版本

举例：

- groupId：com.atguigu.maven
- artifactId：pro01-atguigu-maven
- version：1.0-SNAPSHOT



#### 1.1.3 坐标和仓库中 jar 包的存储路径之间的对应关系

坐标：

```xml
<groupId>javax.servlet</groupId>
<artifactId>servlet-api</artifactId>
<version>2.5</versio
```

上面坐标对应的 jar 包在 Maven 本地仓库中的位置：

```
Maven本地仓库根目录\javax\servlet\servlet-api\2.5\servlet-api-2.5.jar
```



### 1.2 实验操作：生成mavne工程

### 1.2.1 创建目录作为后面操作的工作空间，例如

`E:\maven-workspace\space`

> 此时我们已经有了三个目录，分别是：
>
> - Maven 核心程序：中军大帐
> - Maven 本地仓库：兵营
> - 本地工作空间：战场

#### 1.2.2 然后在命令行进入此目录

![image-20221022212147284](03_使用maven：命令行环境/image-20221022212147284.png)



#### 1.2.3 运行 `mvn archetype:generate` 命令

![img](03_使用maven：命令行环境/img008.be45c9ad.png)



```shell
下面根据提示操作

mvn archetype:generate

Choose a number or apply filter (format: [groupId:]artifactId, case sensitive contains): 7:【直接回车，使用默认值】

Define value for property 'groupId': com.atguigu.maven

Define value for property 'artifactId': pro01-maven-java

Define value for property 'version' 1.0-SNAPSHOT: :【直接回车，使用默认值】

Define value for property 'package' com.atguigu.maven: :【直接回车，使用默认值】

Confirm properties configuration: groupId: com.atguigu.maven artifactId: pro01-maven-java version: 1.0-SNAPSHOT package: com.atguigu.maven Y: :【直接回车，表示确认。如果前面有输入错误，想要重新输入，则输入 N 再回车。】
```

然后就看到创建出了工程

![image-20221022212910958](03_使用maven：命令行环境/image-20221022212910958.png)

#### 1.2.4 调整

Maven 默认生成的工程，对 junit 依赖的是较低的 3.8.1 版本，我们可以改成较适合的 4.12 版本。

自动生成的 App.java 和 AppTest.java 可以删除。

```xml
<!-- 依赖信息配置 -->
<!-- dependencies复数标签：里面包含dependency单数标签 -->
<dependencies>
	<!-- dependency单数标签：配置一个具体的依赖 -->
	<dependency>
		<!-- 通过坐标来依赖其他jar包 -->
		<groupId>junit</groupId>
		<artifactId>junit</artifactId>
		<version>4.12</version>
		
		<!-- 依赖的范围 -->
		<scope>test</scope>
	</dependency>
</dependencies>
```

#### 1.2.5 自动生成的 pom.xml 解读

```xml
<!-- 当前Maven工程的坐标 -->
<groupId>com.atguigu.maven</groupId>
<artifactId>pro01-maven-java</artifactId>
<version>1.0-SNAPSHOT</version>

<!-- 当前Maven工程的打包方式，可选值有下面三种： -->
<!-- jar：表示这个工程是一个Java工程  -->
<!-- war：表示这个工程是一个Web工程 -->
<!-- pom：表示这个工程是“管理其他工程”的工程 -->
<packaging>jar</packaging>

<name>pro01-maven-java</name>
<url>http://maven.apache.org</url>

<properties>
    <!-- 工程构建过程中读取源码时使用的字符集 -->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>

<!-- 当前工程所依赖的jar包 -->
<dependencies>
    <!-- 使用dependency配置一个具体的依赖 -->
    <dependency>

        <!-- 在dependency标签内使用具体的坐标依赖我们需要的一个jar包 -->
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>

        <!-- scope标签配置依赖的范围 -->
        <scope>test</scope>
    </dependency>
</dependencies>
```



### 1.3 Maven核心概念：POM

#### 1.3.1 含义

POM：**P**roject **O**bject **M**odel，项目对象模型。和 POM 类似的是：DOM（Document Object Model），文档对象模型。它们都是模型化思想的具体体现。

#### 1.3.2 模型化思想

POM 表示将工程抽象为一个模型，再用程序中的对象来描述这个模型。这样我们就可以用程序来管理项目了。我们在开发过程中，最基本的做法就是将现实生活中的事物抽象为模型，然后封装模型相关的数据作为一个对象，这样就可以在程序中计算与现实事物相关的数据。

#### 1.3.3 对应的配置文件

POM 理念集中体现在 Maven 工程根目录下 **pom.xml** 这个配置文件中。所以这个 pom.xml 配置文件就是 Maven 工程的核心配置文件。其实学习 Maven 就是学这个文件怎么配置，各个配置有什么用。



### 1.4 Maven核心概念：约定的目录结构

#### 1.4.1 各个目录的作用

![image-20221022214531440](03_使用maven：命令行环境/image-20221022214531440.png)

另外还有一个 target 目录专门存放构建操作输出的结果。

#### 1.4.2 约定目录结构的意义

Maven 为了让构建过程能够尽可能自动化完成，所以必须约定目录结构的作用。例如：Maven 执行编译操作，必须先去 Java 源程序目录读取 Java 源代码，然后执行编译，最后把编译结果存放在 target 目录。

#### 1.4.3 约定大于配置

Maven 对于目录结构这个问题，没有采用配置的方式，而是基于约定。这样会让我们在开发过程中非常方便。如果每次创建 Maven 工程后，还需要针对各个目录的位置进行详细的配置，那肯定非常麻烦。

目前开发领域的技术发展趋势就是：约定大于配置，配置大于编码。



## 2. 在 Maven 工程中编写代码

### 2.1 主体程序

![image-20221022215013157](03_使用maven：命令行环境/image-20221022215013157.png)

主体程序指的是被测试的程序，同时也是将来在项目中真正要使用的程序。

```java
package com.atguigu.maven;

public class Calculator {
    public int sum(int i, int j){
        return i + j;
    }
}
```

### 2.2 测试程序

![image-20221022215140938](03_使用maven：命令行环境/image-20221022215140938.png)

```java
package com.atguigu.maven;

import org.junit.Test;
import com.atguigu.maven.Calculator;

// 静态导入的效果是将Assert类中的静态资源导入当前类
// 这样一来，在当前类中就可以直接使用Assert类中的静态资源，不需要写类名
import static org.junit.Assert.*;

public class CalculatorTest{

    @Test
    public void testSum(){

        // 1.创建Calculator对象
        Calculator calculator = new Calculator();

        // 2.调用Calculator对象的方法，获取到程序运行实际的结果
        int actualResult = calculator.sum(5, 3);

        // 3.声明一个变量，表示程序运行期待的结果
        int expectedResult = 8;

        // 4.使用断言来判断实际结果和期待结果是否一致
        // 如果一致：测试通过，不会抛出异常
        // 如果不一致：抛出异常，测试失败
        assertEquals(expectedResult, actualResult);
    }
}
```



## 3. 执行Maven的构建命令

### 3.1 要求

运行 Maven 中和构建操作相关的命令时，必须进入到 pom.xml 所在的目录。如果没有在 pom.xml 所在的目录运行 Maven 的构建命令，那么会看到下面的错误信息：

```
The goal you specified requires a project to execute but there is no POM in this directory
```

> mvn -v 命令和构建操作无关，只要正确配置了 PATH，在任何目录下执行都可以。而构建相关的命令要在 pom.xml 所在目录下运行——操作哪个工程，就进入这个工程的 pom.xml 目录。

### 3.2 清理操作

mvn clean

效果：删除 target 目录

### 3.3 编译操作

主程序编译：mvn compile

测试程序编译：mvn test-compile

主体程序编译结果存放的目录：target/classes

测试程序编译结果存放的目录：target/test-classes

### 3.4 测试操作

mvn test

测试的报告存放的目录：target/surefire-reports

### 3.5 打包操作

mvn package

打包的结果——jar 包，存放的目录：target

### 3.6 安装操作

mvn install

安装的效果是将本地构建过程中生成的 jar 包存入 Maven 本地仓库。这个 jar 包在 Maven 仓库中的路径是根据它的坐标生成的。

```xml
<groupId>com.atguigu.maven</groupId>
<artifactId>pro01-maven-java</artifactId>
<version>1.0-SNAPSHOT</version>
```

在 Maven 仓库中生成的路径如下：

```
D:\maven-rep1026\com\atguigu\maven\pro01-maven-java\1.0-SNAPSHOT\pro01-maven-java-1.0-SNAPSHOT.jar
```

另外，安装操作还会将 pom.xml 文件转换为 XXX.pom 文件一起存入本地仓库。所以我们在 Maven 的本地仓库中想看一个 jar 包原始的 pom.xml 文件时，查看对应 XXX.pom 文件即可，它们是名字发生了改变，本质上是同一个文件。



## 4. 创建Maven版Web工程

### 4.1 说明

使用 mvn archetype:generate 命令生成 Web 工程时，需要使用一个专门的 archetype。这个专门生成 Web 工程骨架的 archetype 可以参照官网看到它的用法：

![img](03_使用maven：命令行环境/img014.942770a3.png)

参数 archetypeGroupId、archetypeArtifactId、archetypeVersion 用来指定现在使用的 maven-archetype-webapp 的坐标。

### 4.2 操作

注意：如果在上一个工程的目录下执行 mvn archetype:generate 命令，那么 Maven 会报错：不能在一个非 pom 的工程下再创建其他工程。所以不要再刚才创建的工程里再创建新的工程，**请回到工作空间根目录**来操作。

然后运行生成工程的命令：

```shell
mvn archetype:generate -DarchetypeGroupId=org.apache.maven.archetypes -DarchetypeArtifactId=maven-archetype-webapp -DarchetypeVersion=1.4
```

下面的操作按照提示执行：

```
Define value for property 'groupId': com.atguigu.maven Define value for property 'artifactId': pro02-maven-web Define value for property 'version' 1.0-SNAPSHOT: :【直接回车，使用默认值】

Define value for property 'package' com.atguigu.maven: :【直接回车，使用默认值】 Confirm properties configuration: groupId: com.atguigu.maven artifactId: pro02-maven-web version: 1.0-SNAPSHOT package: com.atguigu.maven Y: :【直接回车，表示确认】
```



### 4.3 生成的pom.xml

```xml
<packaging>war</packaging>
```

### 4.4 生成的Web工程的目录结构

![image-20221022222307518](03_使用maven：命令行环境/image-20221022222307518.png)

webapp 目录下有 index.jsp

WEB-INF 目录下有 web.xml

### 4.5  为这个工程编写简单代码

在 main 目录下创建 java 目录

![image-20221022222403425](03_使用maven：命令行环境/image-20221022222403425.png)

在 java 目录下创建 Servlet 类所在的包的目录

![image-20221022222418025](03_使用maven：命令行环境/image-20221022222418025.png)

在包下创建 Servlet 类

```java
package com.atguigu.maven;

import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.ServletException;
import java.io.IOException;

public class HelloServlet extends HttpServlet{

    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        response.getWriter().write("hello maven web");
    }
}
```

在 web.xml 中注册 Servlet

```xml
<servlet>
    <servlet-name>helloServlet</servlet-name>
    <servlet-class>com.atguigu.maven.HelloServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>helloServlet</servlet-name>
    <url-pattern>/helloServlet</url-pattern>
</servlet-mapping>
```

在 index.jsp 页面编写超链接

```xml
<html>
    <body>
        <h2>Hello World!</h2>
        <a href="helloServlet">Access Servlet</a>
    </body>
</html>
```

### 4.6 编译

此时直接执行 mvn compile 命令出错：

```
DANGER

程序包 javax.servlet.http 不存在

程序包 javax.servlet 不存在

找不到符号

符号: 类 HttpServlet

……
```

上面的错误信息说明：我们的 Web 工程用到了 HttpServlet 这个类，而 HttpServlet 这个类属于 servlet-api.jar 这个 jar 包。此时我们说，Web 工程需要依赖 servlet-api.jar 包。

![image-20221022222541569](03_使用maven：命令行环境/image-20221022222541569.png)



### 4.7 配置对 servlet-api.jar 包的依赖

maven仓库查询网站：https://mvnrepository.com/ 使用关键词搜索，然后在搜索结果列表中选择适合的使用。

比如，我们找到的 servlet-api 的依赖信息：

```xml
<!-- https://mvnrepository.com/artifact/javax.servlet/javax.servlet-api -->
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>
```

这样就可以把上面的信息加入 pom.xml。重新执行 mvn compile 命令

### 4.8 将 Web 工程打包为 war 包

运行 mvn package 命令，生成 war 包的位置如下图所示：

![image-20221022222649421](03_使用maven：命令行环境/image-20221022222649421.png)

### 4.9 将 war 包部署到 Tomcat 上运行

将 war 包复制到 Tomcat/webapps 目录下

![image-20221022222721444](03_使用maven：命令行环境/image-20221022222721444.png)

启动 Tomcat：

![image-20221022222733082](03_使用maven：命令行环境/image-20221022222733082.png)

![image-20221022222740819](03_使用maven：命令行环境/image-20221022222740819.png)

通过浏览器尝试访问：http://localhost:8080/pro02-maven-web/index.jsp





## 5. 让Web工程依赖Java工程

### 5.1 观念

明确一个意识：从来只有 Web 工程依赖 Java 工程，没有反过来 Java 工程依赖 Web 工程。本质上来说，Web 工程依赖的 Java 工程其实就是 Web 工程里导入的 jar 包。最终 Java 工程会变成 jar 包，放在 Web 工程的 WEB-INF/lib 目录下。

### 5.2 操作

在 pro02-maven-web 工程的 pom.xml 中，找到 dependencies 标签，在 dependencies 标签中做如下配置：

```xml
<!-- 配置对Java工程pro01-maven-java的依赖 -->
<!-- 具体的配置方式：在dependency标签内使用坐标实现依赖 -->
<dependency>
	<groupId>com.atguigu.maven</groupId>
	<artifactId>pro01-maven-java</artifactId>
	<version>1.0-SNAPSHOT</version>
</dependency>
```



### 5.3 在Web工程中，编写测试代码

1. 补充创建目录

   pro02-maven-web\src\test\java\com\atguigu\maven

2. 确认 Web 工程依赖了 junit

   ```xml
   <dependency>
       <groupId>junit</groupId>
       <artifactId>junit</artifactId>
       <version>4.12</version>
       <scope>test</scope>
   </dependency>
   ```

3. 创建测试类

   把 Java 工程的 CalculatorTest.java 类复制到 pro02-maven-wb\src\test\java\com\atguigu\maven 目录下

### 5.4 执行Maven命令

在这个web项目下分别执行以下命令

```shell
mvn test
# 说明：测试操作中会提前自动执行编译操作，测试成功就说明编译也是成功的。

mvn package
```

![img](03_使用maven：命令行环境/img024.91b00e04.png)

通过查看 war 包内的结构，我们看到被 Web 工程依赖的 Java 工程确实是会变成 Web 工程的 WEB-INF/lib 目录下的 jar 包。

![image-20221023104140604](03_使用maven：命令行环境/image-20221023104140604.png)

### 5.5 查看当前工程依赖

查看当前 Web 工程所依赖的 jar 包的列表

```shell
mvn dependency:list

[INFO] The following files have been resolved:
[INFO] org.hamcrest:hamcrest-core:jar:1.3:test
[INFO] javax.servlet:javax.servlet-api:jar:3.1.0:provided
[INFO] com.atguigu.maven:pro01-maven-java:jar:1.0-SNAPSHOT:compile
[INFO] junit:junit:jar:4.12:test
```

以树形结构查看当前 Web 工程的依赖信息

```shell
mvn dependency:tree

[INFO] com.atguigu.maven:pro02-maven-web:war:1.0-SNAPSHOT
[INFO] +- junit:junit:jar:4.12:test
[INFO] | \- org.hamcrest:hamcrest-core:jar:1.3:test
[INFO] +- javax.servlet:javax.servlet-api:jar:3.1.0:provided
[INFO] \- com.atguigu.maven:pro01-maven-java:jar:1.0-SNAPSHOT:compile
```

我们在 pom.xml 中并没有依赖 hamcrest-core，但是它却被加入了我们依赖的列表。原因是：junit 依赖了hamcrest-core，然后基于依赖的传递性，hamcrest-core 被传递到我们的工程了。



## 6. 依赖的范围

直接上结论：

compile：通常使用的第三方框架的 jar 包这样在项目实际运行时真正要用到的 jar 包都是以 compile 范围进行依赖的。比如 SSM 框架所需jar包。

test：测试过程中使用的 jar 包，以 test 范围依赖进来。比如 junit。

provided：在开发过程中需要用到的“服务器上的 jar 包”通常以 provided 范围依赖进来。比如 servlet-api、jsp-api。而这个范围的 jar 包之所以不参与部署、不放进 war 包，就是避免和服务器上已有的同类 jar 包产生冲突，同时减轻服务器的负担。说白了就是：“**服务器上已经有了，你就别带啦！**”



## 7. 依赖的传递性

直接上结论：

complie：可以进行依赖传递

test/provided ：不可以进行依赖传递



## 8. 依赖的排除

当 A 依赖 B，B 依赖 C 而且 C 可以传递到 A 的时候，A 不想要 C，需要在 A 里面把 C 排除掉。而往往这种情况都是为了避免 jar 包之间的冲突。

![image-20221023111610523](03_使用maven：命令行环境/image-20221023111610523.png)

所以配置依赖的排除其实就是阻止某些 jar 包的传递。因为这样的 jar 包传递过来会和其他 jar 包冲突。

配置方式：

```xml
<dependency>
    <groupId>com.atguigu.maven</groupId>
    <artifactId>pro01-maven-java</artifactId>
    <version>1.0-SNAPSHOT</version>
    <scope>compile</scope>
    <!-- 使用excludes标签配置依赖的排除	-->
    <exclusions>
        <!-- 在exclude标签中配置一个具体的排除 -->
        <exclusion>
            <!-- 指定要排除的依赖的坐标（不需要写version） -->
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```



## 9. 继承

### 9.1 概念

Maven工程之间，A 工程继承 B 工程

- B 工程：父工程
- A 工程：子工程

本质上是 A 工程的 pom.xml 中的配置继承了 B 工程中 pom.xml 的配置。

### 9.2 作用

在父工程中统一管理项目中的依赖信息，具体来说是管理依赖信息的版本。

它的背景是：

- 对一个比较大型的项目进行了模块拆分。
- 一个 project 下面，创建了很多个 module。
- 每一个 module 都需要配置自己的依赖信息。

它背后的需求是：

- 在每一个 module 中各自维护各自的依赖信息很容易发生出入，不易统一管理。
- 使用同一个框架内的不同 jar 包，它们应该是同一个版本，所以整个项目中使用的框架版本需要统一。
- 使用框架时所需要的 jar 包组合（或者说依赖信息组合）需要经过长期摸索和反复调试，最终确定一个可用组合。这个耗费很大精力总结出来的方案不应该在新的项目中重新摸索。

通过在父工程中为整个项目维护依赖信息的组合既**保证了整个项目使用规范、准确的 jar 包**；又能够将**以往的经验沉淀**下来，节约时间和精力。

### 9.3 举例

在一个工程中依赖多个 Spring 的 jar 包

>[INFO] +- org.springframework:**spring-core**:jar:**4.0.0**.RELEASE:compile
>[INFO] | \- commons-logging:commons-logging:jar:1.1.1:compile
>[INFO] +- org.springframework:**spring-beans**:jar:**4.0.0**.RELEASE:compile
>[INFO] +- org.springframework:**spring-context**:jar:**4.0.0**.RELEASE:compile
>[INFO] +- org.springframework:**spring-expression**:jar:4.0.0.RELEASE:compile
>[INFO] +- org.springframework:**spring-aop**:jar:**4.0.0**.RELEASE:compile
>[INFO] | \- aopalliance:aopalliance:jar:1.0:compile

使用 Spring 时要求所有 Spring 自己的 jar 包版本必须一致。为了能够对这些 jar 包的版本进行统一管理，我们使用继承这个机制，将所有版本信息统一在父工程中进行管理。



### 9.4 操作

#### 9.4.1 创建父工程

创建的过程和前面创建 pro01-maven-java 一样。

工程名称：pro03-maven-parent

工程创建好之后，要修改它的打包方式：

```xml
<groupId>com.atguigu.maven</groupId>
<artifactId>pro03-maven-parent</artifactId>
<version>1.0-SNAPSHOT</version>

<!-- 当前工程作为父工程，它要去管理子工程，所以打包方式必须是 pom -->
<packaging>pom</packaging>
```

只有打包方式为 pom 的 Maven 工程能够管理其他 Maven 工程。打包方式为 pom 的 Maven 工程中不写业务代码，它是专门管理其他 Maven 工程的工程。

#### 9.4.2 创建模块工程

模块工程类似于 IDEA 中的 module，所以需要**进入 pro03-maven-parent 工程的根目录**，然后运行 mvn archetype:generate 命令来创建模块工程。

假设，我们创建三个模块工程：

![image-20221023114221486](03_使用maven：命令行环境/image-20221023114221486.png)

#### 9.4.3 查看被添加新内容的父工程 pom.xml

下面 modules 和 module 标签是聚合功能的配置

```xml
<modules>  
    <module>pro04-maven-module</module>
    <module>pro05-maven-module</module>
    <module>pro06-maven-module</module>
</modules>
```



#### 9.4.4 解读子工程的pom.xml

```xml
<!-- 使用parent标签指定当前工程的父工程 -->
<parent>
    <!-- 父工程的坐标 -->
    <groupId>com.atguigu.maven</groupId>
    <artifactId>pro03-maven-parent</artifactId>
    <version>1.0-SNAPSHOT</version>
</parent>

<!-- 子工程的坐标 -->
<!-- 如果子工程坐标中的groupId和version与父工程一致，那么可以省略 -->
<!-- <groupId>com.atguigu.maven</groupId> -->
<artifactId>pro04-maven-module</artifactId>
<!-- <version>1.0-SNAPSHOT</version> -->
```



#### 9.4.5 在父工程中配置依赖的统一管理

```xml
<!-- 使用dependencyManagement标签配置对依赖的管理 -->
<!-- 被管理的依赖并没有真正被引入到工程 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>4.0.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>4.0.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>4.0.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-expression</artifactId>
            <version>4.0.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aop</artifactId>
            <version>4.0.0.RELEASE</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```



#### 9.4.6 子工程中引用那些被父工程管理的依赖

关键点：省略版本号

```xml
<!-- 子工程引用父工程中的依赖信息时，可以把版本号去掉。	-->
<!-- 把版本号去掉就表示子工程中这个依赖的版本由父工程决定。 -->
<!-- 具体来说是由父工程的dependencyManagement来决定。 -->
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-beans</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-expression</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
    </dependency>
</dependencies>
```



#### 9.4.7 在父工程中升级依赖信息的版本

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>4.1.4.RELEASE</version>
</dependency>
```

然后在子工程中运行mvn dependency:list，效果如下：

> [INFO] org.springframework:spring-aop:jar:4.1.4.RELEASE:compile
> [INFO] org.springframework:spring-core:jar:4.1.4.RELEASE:compile
> [INFO] org.springframework:spring-context:jar:4.1.4.RELEASE:compile
> [INFO] org.springframework:spring-beans:jar:4.1.4.RELEASE:compile
> [INFO] org.springframework:spring-expression:jar:4.1.4.RELEASE:compile

#### 9.4.8 在父工程中声明自定义属性

```xml
<!-- 通过自定义属性，统一指定Spring的版本 -->
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

    <!-- 自定义标签，维护Spring版本数据 -->
    <atguigu.spring.version>4.3.6.RELEASE</atguigu.spring.version>
</properties>
```

在需要的地方使用`${}`的形式来引用自定义的属性名：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>${atguigu.spring.version}</version>
</dependency>
```

真正实现“一处修改，处处生效”。

### 9.5 实际意义

![image-20221023114435000](03_使用maven：命令行环境/image-20221023114435000.png)

编写一套符合要求、开发各种功能都能正常工作的依赖组合并不容易。如果公司里已经有人总结了成熟的组合方案，那么再开发新项目时，如果不使用原有的积累，而是重新摸索，会浪费大量的时间。为了提高效率，我们可以使用工程继承的机制，让成熟的依赖组合方案能够保留下来。

如上图所示，公司级的父工程中管理的就是成熟的依赖组合方案，各个新项目、子系统各取所需即可。



## 10. 聚合

### 10.1 聚合含义

部分组成整体

### 10.2 maven中的聚合

使用一个“总工程”将各个“模块工程”汇集起来，作为一个整体对应完整的项目。

- 项目：整体
- 模块：部分

> 概念的对应关系：
>
> 从继承关系角度来看：
>
> - 父工程
> - 子工程
>
> 从聚合关系角度来看：
>
> - 总工程
> - 模块工程



### 10.3 好处

- 一键执行 Maven 命令：很多构建命令都可以在“**总工程**”中一键执行。

  以 mvn install 命令为例：Maven 要求**有父工程时先安装父工程；有依赖的工程时，先安装被依赖的工程。**我们自己考虑这些规则会很麻烦。但是工程聚合之后，在总工程执行 mvn install 可以一键完成安装，而且会自动按照正确的顺序执行。

- 配置聚合之后，各个模块工程会在总工程中展示一个列表，让项目中的各个模块一目了然。



### 10.4 聚合的配置

在总工程中配置 modules 即可：

```xml
<modules>  
    <module>pro04-maven-module</module>
    <module>pro05-maven-module</module>
    <module>pro06-maven-module</module>
</modules>
```



### 10.5 依赖循环问题

如果 A 工程依赖 B 工程，B 工程依赖 C 工程，C 工程又反过来依赖 A 工程，那么在执行构建操作时会报下面的错误：

```
[ERROR] [ERROR] The projects in the reactor contain a cyclic reference:
```

这个错误的含义是：循环引用。



## 11. 聚合和父子工程功能辨析

父子工程的设计是为了统一管理版本，以子工程的标签`<parent>` 体现

聚合工程的设计是为了管理多依赖之间打包顺序，自动帮我们敲定打包顺序，以总工程的`<modules>`标签体现





