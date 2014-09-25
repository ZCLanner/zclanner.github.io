title: 《Spring源码深入解析》读书笔记：Bean是如何被加载到内存中的
date: 2014-09-24 18:52:05
tags: [Java, Spring]
---

阅读开源代码对我这样的新手是一件很头疼的事儿，在作者写代码的时候，他首先会假设一个代码使用场景，然后根据场景进行分析，最后才是架构设计和实现；但是在阅读代码的时候，读者只能从代码局部入手，逐步分析架构设计，最后才能从整体上把握作者解决的究竟是怎样一个问题。但这种方法很容易纠结于代码细节无法自拔，一种情况是陷入了代码的海洋，跟着函数调用跳来跳去最后不知道自己走到哪里；另一种情况是会遇到一些奇怪的实现，这些奇怪的实现本是代码作者在一些特殊场景下的解决方案，但由于这种特殊情况很难想象也十分罕见，读者对此无法理解，自然导致没有办法读懂这些奇怪的设计。所以，有一本书或者有一个人能高屋建瓴的讲解代码解决的问题和解决思路，比起硬读代码效率要高很多。

之前[亮哥](http://weibo.com/wully)推荐了一本《Spring技术内幕》，读起来跟看代码差不多；偶然又看到了这本《Spring源码深入解析》，抱着试试看的态度买了一本，读了四章还不错。现在做个笔记下来，为下一个半年的提升KPI做准备。

<!-- more -->

这本书的第一部分一共七章，讲的是Bean加载、容器扩展功能（也就是最常用的ApplicationContext）和Spring AOP的实现，刚刚阅读的前四章讲的是Bean加载的准备工作——把Bean的定义加载到内存中，为getBean做准备。这部分的工作简单说可以概括为一句话：把定义在Resource中的Java
Bean定义通过BeanDefinitionReader，初始化为BeanDefinition并注册到BeanDefinitionRegistry中；再进一步，想必getBean方法的实现各位也了解了，无非根据Bean的名字或类行到BeanDefinitionRegistry里面找到相应的BeanDefinition，然后根据BeanDefinition各成员的值反射成真正的Bean供调用者使用。下面以最常见的XmlBeanFactory为例，看看究竟是怎么实现的。

如何入手？
---
读源码，最好的方式是单步跟踪，一步一步的调试，在调试过程中观察代码的跳转逻辑和变量的变化，这样的方法比枯燥的读源码来得直观得多。那么，要想跟踪Spring源码，怎么办呢？很简单，只需要写两行代码就行了。

```java
ClassPathResource resource = new ClassPathResource("applicationContext.xml");
BeanFactory bf = new XmlBeanFactory(resource);
```
