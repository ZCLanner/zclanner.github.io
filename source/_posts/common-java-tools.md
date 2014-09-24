title: java的几个命令行工具备忘
date: 2014-09-24 18:58:31
tags: Java
---

开始这篇博客之前，请先装个JDK，百度、google、bing都能帮到你，怎么装JDK我就不在这罗嗦了。

先写个Java的Hello World
====

写代码
----

一切都是从Hello World开始的，Java的Hello World很简单，创建一个叫Hello.java的源码文件，然后把这些代码放进去

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello World!!!\n");
    }
}
```

那么现在的目录结构就是这样的：

```
./
└── Hello.java
```

编译
----

很简单，只有一个叫Hello的java源码文件。在该目录下执行`javac Hello.java`，即对源码进行编译，目录结构就变成了这样：

```
./
├── Hello.class
└── Hello.java
```

多了一个叫后缀为class的文件。javac为java的编译器，它把java源码文件编译成字节文件.class，这个class文件可以直接在java虚拟机中执行。

运行
----

若想将class文件加载进java虚拟机并运行的话，执行`java Hello`即可。这有个坑，java后面跟的是类名`Hello`，而不是字节文件名`Hello.class`。接下来，Hello World就看见了。

```
Hello World!!!
```

总结
----

 在这小结的最后，来用一张图总结一下上面都讲了什么

```
.
                                                       +-----------------+
+------------+               +-----------+             |+-----------+    |
| Hello.java |  == javac ==> |Hello.class| == java ==> ||Hello.class| JVM|
+------------+               +-----------+             |+-----------+    |
                                                       +-----------------+
```

<!-- more -->

添加一点小需求
===

描述需求
----

Hello world只是一个最简单的场景，下面把场景稍微复杂一点儿。现在有一个需求：

1. 用户输入用户名密码
2. 根据得到的用户名密码进行验证
3. 验证结束后，根据验证结果提示用户：
    * 若验证成功，提示用户登录成功
    * 若验证失败，提示用户重新输入

写代码
----

很简单的一个小需求，但是为了看起来nb一些，要给他设计一个复杂的架构，这样看起来才能更高端。下面是类图：

![类图](http://lanner-blog-images.u.qiniudn.com/java-web-hello-01.png)

简单说一下上面三个类都是干嘛的

* UserBean：两个成员变量用户名userName和密码password，用来记录登录信息的一个java bean，除了setter和getter外，它还有一个方法isEqualTo来判断两个UserBean是否相等
* LoginValidator：用于验证传入的userBean是否有效的，它有一个默认的defaultUserBean，LoginValidator会把传入的userBean与defaultUserBean进行比较，返回结果。
* Shell：负责采集用户输入，打印提示信息的类，程序的入口

接下来，建立一个目录结构，java有一个package的概念，是为了方便的管理源码文件。其实在编译时寻找类的基本规则，就是把包名转化为目录名，再按照目录结构依次查找。比如在编译Shell.java时引用到了me.lanner.test.placeholdertest.UserBean，那么javac会以系统的CLASSPATH环境变量为根目录，在这个根目录中寻找me/lanner/test/placehodertest/UserBean.class，若不存在这个类，javac会寻找CLASSPATH下的me/lanner/test/placehodertest/UserBean.java，并把它编译为UserBean.class。

在这里多说一句，关于CLASSPATH的具体内容是什么，不同的工具有不同的定义，像我这种java初学者，一定是根据网上的最通用的配置方法，把CLASSPATH设为`.;%JAVA_HOME%\lib;%JAVA_HOME%\lib\tools.jar`；如果有maven的使用经验的话，一定会知道maven的pom文件中，有对dependency的定义，maven会把自己的本地仓库也加到CLASSPATH中；同样，在使用eclipse时，常常会添加第三方jar包，这一样是做了一个修改编译时CLASSPATH的操作，以便在编译时可以查找到相应的类。

下面，来看看建立的目录结构是什么样的。

```
.
└─me
    └─lanner
        └─test
            └─placeholdertest
                └─LoginValidator.java
                └─Shell.java
                └─UserBean.java
```

以上三个类均属于me.lanner.test.placeholdertest这个包，所以相应的也把它们都放在me/lanner/test/placeholdertest/这个目录下面，至于源码的内容，请看源码清单一节，这里就不多说了。

编译
----

下面开始编译，编译很简单，进入到根目录，也就是me的上一层目录，然后输入

```shell
javac me/lanner/test/placeholdertest/Shell.java
```

再来看看目录结构变成了什么样子

```
.
└─me
    └─lanner
        └─test
            └─placeholdertest
                └─LoginValidator.class
                └─LoginValidator.java
                └─Shell.class
                └─Shell.java
                └─UserBean.class
                └─UserBean.java
```

多了3个与java文件对应的class文件，它们也是编译的副产品。由于Shell.java中引用到了UserBean和LoginValidator，javac会按照上文中叙述的规则去查找相关的类，如果没有找到，则会找到相应的java文件并把它编译为class文件。以查找UserBean为例，当前系统中CLASSPATH的设置为`.;%JAVA_HOME%\lib;%JAVA_HOME%\lib\tools.jar`，注意，CLASSPATH实际上是三个值`.`、`%JAVA_HOME\lib%`、`%JAVA_HOME%\lib\tools.jar`，javac会分别去查找`./me/lanner/test/placeholdertest/UserBean.class`、`%JAVA_HOME%/lib/me/lanner/test/placeholdertest/UserBean.class`和`%JAVA_HOME%/lib/tools.jar!/me/lanner/test/placeholdertest/UserBean.class`，这里面的tools.jar是一个zip压缩包，javac会查看这个zip包里有没有相应的类。很幸运，javac在查找第一个目录`./me/lanner/test/placeholdertest/UserBean.class`时就成功找到了UserBean.java，并编译成了UserBean.class，LoginValidator.class同理。

运行
----

运行Shell.class时，需要输入Shell类的包名加类名

```sh
java me.lanner.test.placeholdertest.Shell
```

结果是登录成功了。

```
Pls enter your username:
lanner
Pls enter your password:
123456
Congratulations for logging in successfully
```

实际上，java在运行时，也会像编译时一样，根据CLASSPATH去查找被引用到的相关类，查找规则一样，不再赘述。

可运行的java包
===

java在发布给其他人使用时，为了减少传输成本，通常会以压缩包的形式发布出去，常见的压缩包类型有jar包、war包和ear包。我还没用过ear包，所以只能讲讲jar包和war包了。

jar
----

jar包是最常见的java发布包格式，其实安装jdk时，java提供了几个默认的jar包，比如tools.jar。tools.jar中打包了一些java中常用的类，提供给码农们使用，是jar包的一种；还有一种jar包是可以直接用java命令行运行的，下面就来说说这种jar包。

制作可执行的jar包有两种方法：使用jar命令、自己压缩。其实使用jar命令就是把自己调整目录并压缩的过程自动化了，为了了解一些细节，这里我只介绍自己调整目录并压缩的方法。

可执行的jar包有两个主要的目录：源码文件目录和配置文件目录。其中源码文件目录从包的顶层开始，比如包名叫me.lanner.test.placeholdertest，那么源码文件目录就是me，me目录下面依次展开；配置文件目录名为META-INF，在这个文件下需要有一个名为MANIFEST.MF的文件，文件中主要指定了主类的类名。看一眼现在的目录结构

```
.
├─me
│  └─lanner
│      └─test
│          └─placeholdertest
│              └─LoginValidator.class
│              └─LoginValidator.java
│              └─Shell.class
│              └─Shell.java
│              └─UserBean.class
│              └─UserBean.java
│
└─META-INF
    └─MANIFEST.MF
```

上面列出的文件中，class文件是必须的，java文件可以删除。

MANIFEST.MF只有一行，指定了主类的类名。

```
Main-Class: me.lanner.test.placeholdertest.Shell

```

在这个配置文件中，需要注意的有两点：

1. Main-Class关键字的冒号后面要有空格
2. 这个配置文件要以空行结束，也就是说在指定了主类之后，还需要加一个回车。我就是在这纠结了好久，比来比去都没发现问题，后来才在百度上找到答案。

实际上，在MANIFEST.MF中，还可以配置运行时的CLASSPATH等其他属性，具体的配置方法各位搜索引擎去查吧，反正有了这个主类的配置，就可以正常运行了。

下面把源码文件夹连同META-INF文件夹，打包成zip包，并把后缀名修改为jar（其实不改也成），然后激动人心的时刻就到来了。请在命令行里输入如下命令（假设这个jar包叫login.jar）：

```
java -jar login.jar
```

war
----

war包是一种用于java web的源码包发布方式，它不能自己运行，需要把它部署到tomcat或jboss等web容器的相应目录下。web容器会根据用户访问的url，通过war包中WEB-INF/web.xml文件的映射规则找到对应的类（通常是一个servlet或listener的子类）来处理相应的请求。

与jar包一样，war包其实也是zip格式的压缩包，只不过这个压缩包的根目录中要包含一个WEB-INF文件夹，这个文件夹下需要一个web.xml文件来配置请求的处理流程。具体如何做才能成功的发布一个war包，今后的博文还会讲到，敬请期待。

源码清单
===

UserBean.java
---

```java
package me.lanner.test.placeholdertest;

public class UserBean {
  private String userName;
  private String password;
  
  public void setUserName(String aUsername) {
    this.userName = aUsername;
  }
  public String getUserName() {
    return this.userName;
  }
  
  public void setPassword(String aPassword) {
    this.password = aPassword;
  }
  public String getPassword() {
    return this.password;
  }

  public boolean isEqualTo(UserBean theUserBean) {
    if (this.userName.equals(theUserBean.userName) && this.password.equals(theUserBean.password)) {
      return true;
    }
    return false;
  }
}
```

LoginValidator.java
---

```java
package me.lanner.test.placeholdertest;

import java.util.Scanner;

public class Shell {

    public static void main(String[] args) {
      boolean ok = false;
      Scanner scanner = new Scanner(System.in);
      String username, password;
      UserBean ub = new UserBean();
      LoginValidator lv = new LoginValidator();
      while (!ok) {
        System.out.println("Pls enter your username:");
        username = scanner.nextLine();
        ub.setUserName(username);
        System.out.println("Pls enter your password:");
        password = scanner.nextLine();
        ub.setPassword(password);
        ok = lv.validate(ub);
        if (ok) {
          System.out.println("Congratulations for logging in successfully");
        } else {
          System.out.println("Sorry, but you entered wrong username or password");
        }
      }
    }
}
```

Shell.java
---

```java
package me.lanner.test.placeholdertest;

public class LoginValidator {

  private static final String USERNAME = "lanner";
  private static final String PASSWORD = "123456";
  private UserBean defaultUserBean;
  public LoginValidator() {
    this.defaultUserBean = new UserBean();
    this.defaultUserBean.setUserName(USERNAME);
    this.defaultUserBean.setPassword(PASSWORD);
  }

  public boolean validate(UserBean userBean) {
    if (this.defaultUserBean == null || userBean == null) {
      return false;
    }
    return this.defaultUserBean.isEqualTo(userBean);
  }
}
```

参考资料
====
* [Java命令行简介](http://developer.51cto.com/art/200510/9902.htm)
* [小雨转晴的专栏 - java 命令行参数](http://blog.csdn.net/wind_324/article/details/6168112)
* [zhengweisincere的博客 - java 打war包，jar包](http://zhengweisincere.blog.163.com/blog/static/498446492011521112812733/)
* [fengyunsky - 打成jar包 在命令行下执行java工程](http://blog.csdn.net/fengyun111999/article/details/5787125)
* [百度百科 - MANIFEST.MF](http://baike.baidu.com/view/1857179.htm)
