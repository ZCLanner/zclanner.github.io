title: 用Maven创建一个war包
date: 2014-09-24 19:04:04
tags: [Java, maven, war, classpath, Spring, bootstrap, velocity]
---

[上回书](2014/09/24/common-java-tools/)说到，可以把war包部署到tomcat或jboss的目录中，来搭建一个网站。war包中最主要的文件就是WEB-INF目录下的web.xml文件，它指定了url的处理规则。那现在就可以YY一下，如果从零开始，搭建一个网站的话，都需要哪些步骤？

1. 写java代码，主要包括一些servlet的子类，也可能会写一些filter
2. 写前段代码，可能是jsp的、可能是velocity的，也可能是单纯的html
3. 引入必要的静态资源，像js、css或者图片等
4. 写web.xml，定义url处理规则
5. 编译java源码
6. 单元测试
7. 建立目录结构，把编译好的class文件、web.xml、前端模板、静态资源按照war包的目录结构组织起来
8. 达成war包并拷贝到tomcat的目录下，重启tomcat

只是YY一下，就需要有8个步骤了，实际操作中可能还会有想不到的意外，像环境问题等。如果工程庞大的话，这八个步骤全部手工操作，难免会有遗漏，这个时候maven就重天而降了。

<!-- more -->

开始使用Maven
====

Maven是什么？我就不套用官方解释了，只说说Maven是干什么的。其实在上一段中已经发现了，上述YY的八个步骤，只有需要动手写代码的几步是必须要完成的，对于编译、单元测试、调整目录结构、打包部署等流程，完全可以按照一个规范来做，从而实现自动化。Maven就是做这个的。Maven的安装、配置和Maven的使用细节，在许晓斌的《Maven实战》中有详细的描述，也可以参考Maven官方的文档来具体学习Maven的使用。这里，我这说说如何用Maven来创建一个war包。

还是以hello为例吧，这里我使用的web容器是tomcat 8.0，并将war包部署在tomcat的webapps目录下，打出的war名字为webtest.war，所以war包部署完毕后，根据tomcat的url规则，在浏览器中访问 http://localhost:8080/webtest/hello 后，可以看见 `Hello` 字样。

为了简单起见，这里使用了eclipse，安装了m2eclipse插件后，eclipse可以新建一个Maven Project。过程中需要选择项目骨架（Archetype），输入groupId、artifactId、Package。我的选择如下：

* Archetype: maven-archetype-webapp 
* groupId: me.lanner.test
* artifactiId: webtest
* package: me.lanner.test.webtest

成功后，在eclipse可以看到项目中包含src/main/resources这样一个文件夹，在Maven的约定中，这个文件夹主要用来存放xml资源文件；java源码文件应该默认存放在src/main/java文件夹中，目前是没有的，需要在工程中新建一个source folder，命名为src/main/java。之后，需要一个servlet来处理url为hello的请求，相应的类为me.lanner.test.webtest.HelloServlet，代码很简单

```java
package me.lanner.test.webtest;

import java.io.*;
import javax.servlet.*;
import javax.servlet.http.*;

public class Hello extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        PrintWriter writer = resp.getWriter();
        writer.println("Hello");
    }
}
```

接下来要修改配置文件webapp/WEB-INF/web.xml，写路由规则，把url为/hello的请求分配给HelloServlet处理，web.xml的源码如下：

```xml
<web-app xmlns="http://java.sun.com/xml/ns/j2ee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee
    http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd"
    version="2.4">
    <servlet>
        <servlet-name>hello</servlet-name>
        <servlet-class>me.lanner.test.webtest.HelloServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>hello</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>
</wep-app>
```
web.xml定义规则可以分两步：
1. 根据servlet类创建一个servlet结点，在这里就是使用me.lanner.test.webtest.HelloServlet类创建了一个名为hello的servlet结点
2. 建立一个映射规则，把url符合条件的请求指向servlet结点，比如把url结尾为/hello的请求指向了`1`中建立的名字为hello的servlet

好了，代码编写工作到此已经结束，下面就是编译、测试、打包、发布，这一系列动作都可以交给maven执行。由于这里并没有写测试代码，而且发布的过程也很简单（其实就是把war包拷过去再重启一下tomcat），因此只需要编译打包就ok了。在根目录，即pom.xml所在目录执行

```
mvn clean package
```

这里顺便说一句，为了编译通过，要在项目中添加servlet包，方式是在pom文件中添加一个dependency结点，结点中指定依赖包的groupId、artifactId和version等。

```xml
<dependencies>
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>3.1.0</version>
    </dependency>
</dependencies>
```

好，上面执行的mvn命令有两条：clean和package，实际上maven做了3件事
1. clean：清理项目，把上次构建时产生的类文件、临时文件等统统干掉
2. compile：编译，把所有的java文件编译为类文件
3. package：打包，把编译好的类文件和其他资源文件如xml等按照war包规则打成war包。

如果一切顺利的话，在项目根目录的target/目录下，应该会生成一个webtest.war，让我们看看类文件是如何被打包为war包的，把war包解压（随便用一个能解析zip包的软件都可以做到），看看war包的目录结构是不是这个样子

```
./
├─META-INF
└─WEB-INF
    ├─ web.xml
    ├─classes
    │  └─me
    │      └─lanner
    │          └─test
    │              └─placeholdertest
    │                   └─Hello.class
    └─lib
         └─javax.servlet-api-3.1.0.jar
```

发现什么规律来了吗？

1. 根目录下有两个文件夹：META-INF和WEB-INF，一切的东西都被搞到了WEB-INF目录中
2. WEB-INF目录下有3部分
    1. web.xml
    2. classes目录，下面存放了被编译的类文件
    3. lib目录，存放了第三方依赖库

也就是说原项目目录中src/main/java中的java文件编译成的class文件都会被拷贝到WEB-INF/classes目录下，实际上，**src/main/resources目录中的xml文件也会被直接拷贝到classes目录下**。如果源码中有classpath引用的话，它实际指向的是哪呢？应该是<font color="red">**classes目录和lib目录**</font>，这句话要牢记，下文会用到。

接下来，把webtest.war拷贝到tomcat的webapps中，重启tomcat，浏览器中访问http://localhost:8080/webtest/hello，看看结果吧。如果失败的话，请移步tomcat的日志文件中差错。

使用前端模板Velocity
===

ok？ok的话开始增加需求，和[上回书](2014/09/24/common-java-tools/)的类似，做一个登录后验证，并给出登录成功/失败信息的网页。架构设计与之前保持一致，验证时采用UserBean + LoginValidator的方式。不过Shell类改为LoginServlet类，用来显示登录页面，若验证通过，跳转到hello页面；验证失败，停留在login页面。**login页面的用户名、密码参数采用POST方法传递。**

开始很简单，新建一个类me.lanner.test.webtest.LoginServlet，在doGet中处理get请求，即展示登录页面；在doPost中获取用户名密码，然后生成UserBean并使用LoginValidator验证。成功，则使用HttpServletResponse的sendRedirect方法跳转到hello页面，代码如下：

```java
package me.lanner.test.webtest;

import ...

public class Login extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        PrintWriter writer = resp.getWriter();
        writer.println(".....");      // 额，这个地方是不是得把Login页面的HTML源码写下来啊？
    }
}

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String username = request.getParameter("username");
        String password = request.getParameter("password");
        if (username != null && username.length() > 0 && password != null && password.length() > 0) {
            UserBean ub = new UserBean();
            ub.setUsername(username);
            ub.setPassword(password);
            LoginValidator lv = new LoginValidator();
            if (!lv.validate(ub)) {
                resp.sendRedirect("hello");    // 验证成功，则跳转到hello页面
            }
        }
    }
}
```

发现了个小问题，在处理get请求时，应该如何把页面给显示出来？之前显示个hello还好，直接用writer写一个hello就好了；但现在我要写个页面啊，难道我要把html的代码用字符串的方式写到java代码里面？不是说好了逻辑和表现分离的吗？是时候引入前端模板了。

所谓的前端模板，就是用来展现前端页面的一个包，里面可能会有少量的逻辑判断，比如在某个位置是要显示一个成功提示还是要显示一个失败提示；也会有for循环来打印许多的html结点。但它的主要任务还是与servlet一道，把请求的处理逻辑和请求处理结果（网页页面）分离开来。Velocity就是一种常见的前端模板，它有一套自己的语法由Velocity包来解析并生成相应的html页面。

要是用Velocity，需要如下几个步骤

1. 引入Velocity依赖包，本文pom中的velocity dependency结点定义为
        
        <dependency>
            <groupId>org.apache.velocity</groupId>
            <artifactId>velocity</artifactId>
            <version>1.7</version>
        </dependency>
        <dependency>
            <groupId>org.apache.velocity</groupId>
            <artifactId>velocity-tools</artifactId>
            <version>2.0</version>
        </dependency>
    
2. 在WEB-INF目录下新建文件velocity.properties，文件内容可以参照下面内容

        resource.loader = webapp
        webapp.resource.loader.class = org.apache.velocity.tools.view.servlet.WebappLoader
        webapp.resource.loader.path = /WEB-INF/vm/
        input.encoding = utf8
        output.encoding = utf8

    其中，webapp.resource.loader.path属性很重要，这个属性值的意义是vm模板文件所在目录，该目录是一个在war包中的相对路径。还记得maven达成的war包的目录结构吗？再帮你回忆一下，war包包括两个目录META-INF和WEB-INF，那么项目目录中的/WEB-INF/vm对应到war包中呢？路径不变，依然是WEB-INF/vm。
3. 在模板目录WEB-INF/vm中新建模板文件，这里需要新建两个模板文件：hello.vm和login.vm，这两个模板文件的内容很简单：
    * hello.vm

        hello

    * login.vm

        <form method="post">
            <h2>Please sign in</h2>
            <input type="text" name="username" placeholder="Email address" required autofocus>
            <small>$usernameError</small>    <!-- $usernameError需要说一说 -->
            <input type="password" name="password" placeholder="Password" required>
            <button type="submit">Sign in</button>
        </form>
        在html中有一个奇怪的东西`$usernameError`，这个东西就是vm里面的一个语法点，它实际上是一个变量，具体内容会根据servlet中放到上下文中的值对应上。这个值怎么设置？请耐心往下看。

4. 处理请求的servlet要使用Velocity包中的VelocityViewServlet子类，而不是采用HttpServlet，实际上，VelocityViewServlet是HttpServlet的子类。并重写handleRequest方法，这个方法合并了HttpServlet的doGet和doPost，需要返回vm模板类Template。这个Template可以用vm模板文件名生成。java代码如下：

        package me.lanner.test.webtest;

        import ...;

        public class Login extends VelocityViewServlet {
            @Override
            protected Template handleRequest(HttpServletRequest request, HttpServletResponse response, Context ctx) {
                String templateName = "login.vm";
                String username = request.getParameter("username");
                String password = request.getParameter("password");
                if (username != null && username.length() > 0 && password != null && password.length() > 0) {    
                    UserBean ub = new UserBean();
                    ub.setUsername(username);
                    ub.setPassword(password);
                    LoginValidator lv = new LoginValidator();
                    if (!lv.validate(ub)) {
                        ctx.put("usernameError", lv.getLastError());    // 注意：设置usernameError
                    } else {
                        templateName = "Hello.vm";
                    }
                }
                return getTemplate(templateName);
            }
        }

    找到了吗？usernameError的值，使用ctx.put函数，放到把键值对放到上下文中，这样vm模板就能取到了。

    打个war包，部署一下再试试，然后访问http://localhost:8080/webtest/login，应该可以看见登录页面并成功登录了。

引入Spring
====

下面我们来接着改进注意到Login的第8行代码了吗？需要new一个UserBean才能进行用户名密码的验证，同样的现象也出现在第11行代码，LoginValidator的生命周期是需要我们手动管理的。仔细想一想，这样的代码可以改成这样：

```java
    public class Login extends VelocityViewServlet {
        private UserBean ub = new UserBean();
        private LoginValidator lv = new LoginValidator();
        
        @Override
        protected Template handleRequest(HttpServletRequest request, HttpServletResponse response, Context ctx) {
            ......
            ......
            this.ub.setUsername(username);
            this.ub.setPassword(password);
            if (this.lv.validate(this.ub)) {
                ......
            }
        }
    }
```
看起来一定程度上精简了代码，也因为省去了每次都初始化的麻烦提高了效率，但是ub和lv的生命周期还是又我们自己管理的。在想想LoginValidator内部的实现，defaultUserBean也是由开发者手动管理的，能不能不用考虑这些java对象的初始化，需要的时候它就在那呢？就好像我们根本不用考虑Login这个Servlet是如何被初始化的，只需要关心它的业务逻辑呢？伟大的Java框架Spring现在就要闪亮登场了！

求助于Bootstrap
====

扩展阅读
====

* [Maven Resources](http://maven.apache.org/articles.html)
* [ITEye - 《Maven实战》样章](http://hzbook.group.iteye.com/group/wiki/2872-Maven-in-action/)

