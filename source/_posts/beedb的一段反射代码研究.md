title: beedb的一段反射代码研究
date: 2014-09-22 22:54:26
tags: GoLang
toc: true
---

```Go
func (orm *Model) ScanPK(output interface{}) *Model {
    if reflect.TypeOf(reflect.Indirect(reflect.ValueOf(output)).Interface()).Kind() == reflect.Slice {
        sliceValue := reflect.Indirect(reflect.ValueOf(output))
        sliceElementType := sliceValue.Type().Elem()
        for i := 0; i &lt; sliceElementType.NumField(); i++ {
            bb := sliceElementType.Field(i).Tag
            if bb.Get("beedb") == "PK" || reflect.ValueOf(bb).String() == "PK" {
                orm.PrimaryKey = sliceElementType.Field(i).Name
            }
        }
    } else {
        tt := reflect.TypeOf(reflect.Indirect(reflect.ValueOf(output)).Interface())
        for i := 0; i &lt; tt.NumField(); i++ {
            bb := tt.Field(i).Tag
            if bb.Get("beedb") == "PK" || reflect.ValueOf(bb).String() == "PK" {
                orm.PrimaryKey = tt.Field(i).Name
            }
        }
    }
    return orm

} 
```

<!-- more -->

开始看谢大的[beedb](https://github.com/astaxie/beedb)，结果在`Model.ScanPk(interface{})`就卡住了，主要是对函数中一些反射相关的API不了解，那就从反射开始研究吧。

先来介绍`ScanPK`中会涉及到的几个数据结构，这里只会写到跟这个函数相关的数据结构的相关字段，如果有兴趣进一步研究的话请移步[golang reflect](http://golang.org/pkg/reflect/)

* Type 表示数据结构类型的数据结构，部分字段的声明如下：  
    
    ```Go
    type Type interface {
        
        // 这个Type的种类
        Kind() Kind

        // 查找这个Type的第 i 个字段
        Field(i int) StructField

        // 字段的数量
        NumField() int
    } 
    ```

* Value 表示数据结构值的数据结构，部分字段的声明如下：

* Kind Go语言内置类型的枚举值，声明如下：

    ```Go
    const (
        Invalid Kind = iota
        Bool
        Int
        Int8
        Int16
        Int32
        Int64
        Uint
        Uint8
        Uint16
        Uint32
        Uint64
        Uintptr
        Float32
        Float64
        Complex64
        Complex128
        Array
        Chan
        Func
        Interface
        Map
        Ptr
        Slice
        String
        Struct
        UnsafePointer
    )
    ```

* StructField 用于描述一个数据结构的字段，部分字段声明如下：

    ```Go
    type StructField struct {
        // 这个字段的名字
        Name    string
        
        Tag       StructTag // 这个字段的注释
    }
    ```

* StructTag StructField中有一个Tag字段，这个字段是对所属字段属性的一种补充。比如现在有一个自定义的数据结构声明如下：
    ```Go
    type Person struct {
        Name string `json:"name"`
        Age int
    }
    ```

与C语言不同的地方是，Name字段在声明后，又添加了一段被反引号(\`)包围的字符串，这个字符串的规则是`键名:"值"`。那么这种类型的结构在被反射时，这段被包围的字符串便会被解析到`StructTag`中，可以使用函数`StructTag.Get(key string)`来获得某个值。

到这里，需要用到的数据结构就都交代完毕了，其中最重要的两个数据结构是`Value`、`Type`，一个表示值，一个表示类型。接下来再看几个函数。

* `func TypeOf(i interface{}) Type`，获得 i 的类型
* `func ValueOf(i interface{}) Value`，获得 i 的值
* `func Indirect(v Value) Value`，获得 v 所指向的内存地址的类型。如果 v 是一个空指针，返回 zero Value；如果 v 不是一个指针，返回 v 自己
* `func (v Value) Interface() (i interface{})`，把 v 转化成一个空接口 interface{}

一切准备就绪，下面开始分析谢大的代码。总体上，这几行代码的思路是查看传入变量`output`的每个字段，如果某个字段带有主键注释的话，那么就把返回的`*Model`的`PrimaryKey`设成这个字段。传入的`output`有两种可能：字典(map)和结构(struct)，对这两种结构，枚举他们的字段有不同的处理方法。

因为我阅读谢大代码的目的是改进这个ORM类库，在新的类库中不想使用字典类，所以跳过第一个分支，直接看第二个分支，看他是如何对struct进行处理的。

1. 使用一连串的反射API获取`output`的类型信息，存储到`tt`中
2. 调用`Type.Field(int)`得到每个字段
3. 查看字段的`Tag`中是否包含了`beedb`键，如果包含了`beedb`键，而且值为`"PK"`，说明这个字段是主键

三步结束，非常完美，令人头疼的就是那一连串的反射API的使用`tt := reflect.TypeOf(reflect.Indirect(reflect.ValueOf(output)).Interface())`。

从简单的开始，如果直接用`reflect.TypeOf(i interface{})`来获得`output`的类型行不行呢？假设有如下的自定义结构

```Go
type Person struct {
    Name string
    Age int
}
```

接着，初始化一个`*Person`，并检查它的类型

```Go
person := &Person{"Lanner", 25}
fmt.Println("Type:", reflect.TypeOf(person).Kind())</pre>
```

输出如下

```Go
Type: *main.Person
```

结果是一个指针，这样便无法接着枚举出它的所有字段了。也就是说，如果只用`reflect.TypeOf(i interface{})`的话，只能保证在传入值是值而不是指针的情况下能正确执行，那么如何保证传入指针的情况下也能执行呢？这就要使用到`reflect.Indirect(v Value)`，允许我在重复一遍这个函数的作用，如果v是一个指针类型的话，它会返回指针v所指向的值的类型。那么代码就可以改进为如下形式

```Go
person := &Person{"Lanner", 25}
fmt.Println("Type:", reflect.TypeOf(reflect.Indirect(reflect.ValueOf(person))))
```

输出如下

```
Type: reflect.Value 
```

是一个Value类型的，还不是想要的Person类型，`Value.Interface()`闪亮登场，由于`reflect.Indirect(v Value)`的返回值永远都是Value，我们对它监测类型没有丝毫意义，所以就要用`Value.Interface()`把它转换为空接口，这样`reflect.TypeOf(i interface{})`就能正确的显示i的类型了。

```Go
person := &Person{"Lanner", 25}
fmt.Println("Type:", reflect.TypeOf(reflect.Indirect(reflect.ValueOf(person)).Interface()))
```

输出如下

```Go
Type: main.Person
```

大功告成！！

*Lanner 水平有限，可能有些地方讲解的不大清楚，感兴趣的读者可以看看下面我给出的参考文献*

* [Go reflect API文档](http://golang.org/pkg/reflect/)
* Go 语言反射的规则([原文](blog.golang.org/laws-of-reflection) [中文译文](http://www.mikespook.com/2011/09/%E5%8F%8D%E5%B0%84%E7%9A%84%E8%A7%84%E5%88%99/))
