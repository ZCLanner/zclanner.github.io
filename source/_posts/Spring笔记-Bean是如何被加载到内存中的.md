title: 《Spring源码深入解析》读书笔记：Bean是如何被加载到内存中的
date: 2014-09-24 18:52:05
tags: [Java, Spring]
---

阅读开源代码对我这样的新手是一件很头疼的事儿，在作者写代码的时候，他首先会假设一个代码使用场景，然后根据场景进行分析，最后才是架构设计和实现；但是在阅读代码的时候，读者只能从代码局部入手，逐步分析架构设计，最后才能从整体上把握作者解决的究竟是怎样一个问题。但这种方法很容易纠结于代码细节无法自拔，一种情况是陷入了代码的海洋，跟着函数调用跳来跳去最后不知道自己走到哪里；另一种情况是会遇到一些奇怪的实现，这些奇怪的实现本是代码作者在一些特殊场景下的解决方案，但由于这种特殊情况很难想象也十分罕见，读者对此无法理解，自然导致没有办法读懂这些奇怪的设计。所以，有一本书或者有一个人能高屋建瓴的讲解代码解决的问题和解决思路，比起硬读代码效率要高很多。

之前[亮哥](http://weibo.com/wully)推荐了一本《Spring技术内幕》，读起来跟看代码差不多；偶然又看到了这本《Spring源码深入解析》，抱着试试看的态度买了一本，读了四章还不错。现在做个笔记下来，为下一个半年的提升KPI做准备。

<!-- more -->

这本书的第一部分一共七章，讲的是Bean加载、容器扩展功能（也就是最常用的ApplicationContext）和Spring AOP的实现，刚刚阅读的前四章讲的是Bean加载的准备工作——把Bean的定义加载到内存中，为getBean做准备。这部分的工作简单说可以概括为一句话：把定义在Resource中的Java Bean定义通过BeanDefinitionReader，初始化为BeanDefinition并注册到BeanDefinitionRegistry中；再进一步，想必getBean方法的实现各位也了解了，无非根据Bean的名字或类行到BeanDefinitionRegistry里面找到相应的BeanDefinition，然后根据BeanDefinition各成员的值反射成真正的Bean供调用者使用。

先解释一下上面的各种英文单词都是干嘛的，再把XmlBeanFactory当个栗子来看看具体是怎么实现的。

* Resource：Spring中对与外部资源的抽象，最常见的是对文件的抽象，特别是XML文件。而且Resource里面通常是保存了Spring使用者的Bean定义，比如applicationContext.xml在被加载时，就会被抽象为Resource来处理。
* BeanDefinitionReader：顾名思义，一个用来读取Resource，并把Resource中保存的Bean定义转化为内存数据结构的类。
* BeanDefinition：一个Bean在XML文件里面的展现形式是`<Bean id="...">...</Bean>`，当这个节点被加载到内存中，就被抽象为BeanDefinition了，在XML Bean节点中的那些关键字，在BeanDefinition中都有相对应的成员变量，感兴趣的可以去看一下源码。如何把一个XML节点转换成BeanDefinition，这个工作自然是由BeanDefinitionReader来完成的。
* BeanDefinitionRegistry：当BeanDefinitionReader完成了转换工作之后，得到的结果保存在哪？接下来的getBean函数，应该到哪去找这些需要的BeanDefinition？这就是BeanDefinitionRegistry的使命，

如何入手？
---
读源码，最好的方式是单步跟踪，一步一步的调试，在调试过程中观察代码的跳转逻辑和变量的变化，这样的方法比枯燥的读源码来得直观得多。那么，要想跟踪Spring源码，怎么办呢？很简单，只需要写两行代码就行了，当然前提是要准备好Spring的配置文件，在这里是放在ClassPath下面的applicationContext.xml。

```java
ClassPathResource resource = new ClassPathResource("applicationContext.xml");
BeanFactory bf = new XmlBeanFactory(resource);
```

有了上面的代码，只要在`new XmlBeanFactory(resource)`一行下一个断点，开始调试，就能跟踪到Spring的代码里面去了。

XmlBeanFactory的初始化
---

下面是XmlBeanFactory在初始化过程中涉及到的类的关系图。图中空心三角加实线代表继承、空心三角加虚线代表实现、实线箭头加虚线代表依赖、实心菱形加实线代表组合。另外由于我用的画图工具没有接口的图例，这里用下划线代表接口，没有下划线的代表类。

![XmlBeanFactory初始化过程中的类图](http://lanner-blog-images.u.qiniudn.com/XmlBeanFactory%E7%B1%BB%E5%9B%BE.png)

有点儿复杂吧，这还是简化之后的东西，各位看起来一定是有点儿晕，麻烦尝试着用点儿时间结合我上文说的大致流程理解一下。利用这段时间，我来画一个面向过程思维的流程图。然后再来说说具体的初始化过程。

![GenericBeanDefinition的诞生](http://lanner-blog-images.u.qiniudn.com/GenericBeanDefinition%E7%9A%84%E8%AF%9E%E7%94%9F.png)

上面的流程图不是没有标准化的图例，是Lanner自己胡乱画的一个东西，为了简单直观的梳理一下一个Bean定义是通过几个步骤被从XML文件的结点中解析出来的。一个方框代表内存里的一个数据结构，一个方框带个尾巴代表一个处理过程，箭头代表一种内存数据结构经过了一个处理过程以后变成了另外一种数据结构。现在开始结合这两张图简单说一下`new XmlBeanFactory(resource)`的时候，究竟发生了什么事儿。

XmlBeanFactory初始化的最终目的是从XML文件中解析出BeanDefinition并放到BeanDefinitionRegistry里面，这里XmlBeanFactory实际上充当了BeanDefinitionRegistry的角色。也就是说，最后的BeanDefinition是以键值対的形式存储在XmlBeanFactory的一个HashMap里面的。

那么在解析出XML文件中的节点是如何变成BeanDefinition的呢？首先要解决文件编码问题，它是UTF8的、还是GBK的、还是其他编码的？为了消除编码的影响，它会被转换成EncodedResource。另一方面，Java中有现成了类库可以解析XML文件，但前提是必须传入Document对象，也就是XML文件在内存中的表示形式，所以接下来EncodedResource会被XmlBeanDefinitionReader的一个成员DocumentLoader转换成Document对象。DefaultBeanDefinitionDocumentReader接到了XmlBeanDefinitionReader传给的Document对象，就可以从中取出一个一个的Element，并把这些Element交给BeanDefinitionParserDelegate或NameSpaceHandler解析成BeanDefition的一个实现GenericBeanDefition，并放到BeanDefinitionRegistry也就是XmlBeanFactory里面。

那么BeanDefitionRegistry是怎么一步一步传给DefaultBeanDefitionDocumentReader的呢？实际上，XmlBeanDefitionReader在初始化时需要一个BeanDefitionRegistry的参数；而它在初始化DefaultBeanDefitionDocumentReader时，又会把BeanDefitionRegistry和NameSpaceHandlerResolver放到XmlReaderContext中传递给DefaultBeanDefinitionDocumentReader。这样最后的真正解析者就能拿到这个Bean的容器了。

自定义XML文件标签的原理
---

上一节提到了两个类NameSpaceHandlerResolver和NameSpaceHandler，实际上NameSpaceHandler是一个接口，它的parse方法是真正把XML节点转换成GenericBeanDefition的关键；而NameSpaceHandlerResolver就是根据当前节点的名字空间返回不同NameSpaceHandler的一个工厂类。它们是这样被使用的：

```Java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                if (delegate.isDefaultNamespace(ele)) {
                    parseDefaultElement(ele, delegate);
                }
                else {
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```

这是DefaultBeanDefinitionReader（XmlBeanDefinitionReader的父类）的一段代码，可以看到如果当前解析的节点的名字空间是默认名字空间（`http://www.springframework.org/schema/beans`）的话，那么就调用解析代理直接解析；否则，调用parseCustomElement。这里注意一下，只要不是beans命名空间，都算是自定义节点，所以像我们平时用到的什么context、aop什么的啊，其实都是用NamespaceHandler解析的。那么，parseCustomElement又是什么呢？

```Java
public BeanDefinition parseCustomElement(Element ele) {
    return parseCustomElement(ele, null);
}

public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
    String namespaceUri = getNamespaceURI(ele);
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if (handler == null) {
        error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
        return null;
    }
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

看到了吧，`this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri)`，也就是根据当前名字空间选择一个自定义的NameSpaceHandler进行自定义节点的解析。他又是根据什么规则返回NameSpaceHandler的呢？

```Java
public NamespaceHandler resolve(String namespaceUri) {
    Map<String, Object> handlerMappings = getHandlerMappings();
    Object handlerOrClassName = handlerMappings.get(namespaceUri);
    ...
    ...
    String className = (String) handlerOrClassName;
    ...
    ...
    Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
    ...
    ...
    NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
    ...
    ...
    return namespaceHandler;
}
    
private Map<String, Object> getHandlerMappings() {
    ...
    ...
    Properties mappings = PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
    ...
    ...
    Map<String, Object> handlerMappings = new ConcurrentHashMap<String, Object>();
    CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
    this.handlerMappings = handlerMappings;
    ...
    return this.handlerMappings;
}
    
    public DefaultNamespaceHandlerResolver(ClassLoader classLoader, String handlerMappingsLocation) {
    Assert.notNull(handlerMappingsLocation, "Handler mappings location must not be null");
    this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
    this.handlerMappingsLocation = handlerMappingsLocation;
}
```

上面是XmlBeanDefinitionReader中用到的NameSpaceHandlerResolver的一个实现DefaultNameSpaceHandlerResolver的几段代码，可以看到在构造函数中记录了一个配置文件位置handlerMappingLocation；然后在resolve函数中读取这个配置文件位置并取出配置信息，也就是名字空间和解析类的映射关系；最后用反射机制初始化这个解析类，并进行解析。

最后一个问题，配置文件位置信息是在哪传进来的？很容易想到，自然是它的调用者XmlBeanDefinitionReader给它的，在哪？

```Java
// XmlBeanDefinitionReader
protected NamespaceHandlerResolver createDefaultNamespaceHandlerResolver() {
    return new DefaultNamespaceHandlerResolver(getResourceLoader().getClassLoader());
}
    
// DefaultNamespaceHandlerResolver
public DefaultNamespaceHandlerResolver(ClassLoader classLoader) {
    this(classLoader, DEFAULT_HANDLER_MAPPINGS_LOCATION);
}
    
public static final String DEFAULT_HANDLER_MAPPINGS_LOCATION = "META-INF/spring.handlers";
```

XmlBeanDefinitionReader在初始化DefaultNamespaceHandlerResolver时没有传入配置文件信息，DefaultNamespaceHandlerResolver就用了默认的位置"META-INF/spring.handlers"，也就是说如果有一些自定义的标签还想把它整合进Spring工程，那么只需要自己实现一个NamespaceHandler，并在自己JAR包下的META-INF/spring.handlers文件中定义名字空间到NamespaceHandler的映射规则即可。具体的方法，网上一搜一大堆，这里就不再赘述了。

感谢各位赏脸能耐着性子看完这条裹脚布，希望能对你阅读Spring代码有些帮助。
