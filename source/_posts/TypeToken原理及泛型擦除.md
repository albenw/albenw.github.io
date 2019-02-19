title: TypeToken原理及泛型擦除
author: alben.wong
abbrlink: a6e1c381
tags:
  - typetoken
  - 泛型
  - 泛型擦除
categories:
  - java
  - guava
date: 2019-02-13 16:36:00
keywords: typetoken 泛型擦除
description: 借助对TypeToken原理的分析，加强对泛型擦除的理解，使得我们能够知道什么时候，通过什么方式可以获取到泛型的类型。
---
## 概要
借助对TypeToken原理的分析，加强对泛型擦除的理解，使得我们能够知道什么时候，通过什么方式可以获取到泛型的类型。

## 泛型擦除
众所周知，Java的泛型只在编译时有效，到了运行时这个泛型类型就会被擦除掉，即`List<String>`和`List<Integer>`在运行时其实都是`List<Object>`类型。

为什么选择这种实现机制？不擦除不行么？  
在Java诞生10年后，才想实现类似于C++模板的概念，即泛型。Java的类库是Java生态中非常宝贵的财富，必须保证向后兼容（即现有的代码和类文件依旧合法）和迁移兼容（泛化的代码和非泛化的代码可互相调用）基于上面这两个背景和考虑，Java设计者采取了“类型擦除”这种折中的实现方式。

同时正正有这个这么“坑”的机制，令到我们无法在运行期间随心所欲的获取到泛型参数的具体类型。


## TypeToken

### 使用
使用过Gson的同学都知道在反序列化时需要定义一个TypeToken类型，像这样
```
private Type type = new TypeToken<List<Map<String, Foo>>>(){}.getType();

//调用fromJson方法时把type传过去，如果type的类型和json保持一致，则可以反序列化出来
gson.fromJson(json, type);

```

### 三个问题

1. 为什么要用TypeToken来定义反序列化的类型？  
正如上面说的，如果直接把List<Map<String, Foo>>的类型传过去，但是因为运行时泛型被擦除了，所以得到的其实是`List<Object>`，那么后面的Gson就不知道要转成`Map<String, Foo>`类型了，这时Gson会默认转成LinkedTreeMap类型。

2. 为什么带有大括号{}？  
这个大括号就是精髓所在。大家都知道，在Java语法中，在这个语境，{}是用来定义匿名类，这个匿名类是继承了TypeToken类，它是TypeToken的子类。

3. 为什么要通过子类来获取泛型的类型？  
这是TypeToken能够获取到泛型类型的关键，这是一个巧妙的方法。  
这个想法是这样子的，既然像`List<String>`这样中的泛型会被擦除掉，那么我用一个子类`SubList extends List<String>`这样的话，在JVM内部中会不会把父类泛型的类型给保存下来呢？我这个子类需要继承的父类的泛型都是已经确定了的呀，果然，JVM是有保存这部分信息的，它是保存在子类的Class信息中，具体看：  
https://stackoverflow.com/questions/937933/where-are-generic-types-stored-in-java-class-files  
那么我们怎么获取这部分信息呢？  
还好，Java有提供API出来：
```java
Type mySuperClass = foo.getClass().getGenericSuperclass();
  Type type = ((ParameterizedType)mySuperClass).getActualTypeArguments()[0];
System.out.println(type);
```

分析一下这段代码，Class类的getGenericSuperClass()方法的注释是：

> Returns the Type representing the direct superclass of the entity (class, interface, primitive type or void) represented by thisClass.  
If the superclass is a parameterized type, the Type object returned must accurately reflect the actual type parameters used in the source code. The parameterized type representing the superclass is created if it had not been created before. See the declaration of ParameterizedType for the semantics of the creation process for parameterized types. If thisClass represents either theObject class, an interface, a primitive type, or void, then null is returned. If this object represents an array class then theClass object representing theObject class is returned

概括来说就是对于带有泛型的class，返回一个ParameterizedType对象，对于Object、接口和原始类型返回null，对于数 组class则是返回Object.class。ParameterizedType是表示带有泛型参数的类型的Java类型，JDK1.5引入了泛型之 后，Java中所有的Class都实现了Type接口，ParameterizedType则是继承了Type接口，所有包含泛型的Class类都会实现 这个接口。

自己调试一下就知道它返回的是什么了。


### 原理
核心的方法就是刚刚说的那两句，剩下的就很简单了。  
我们看看TypeToken的getType方法
```
public final Type getType() {
	//直接返回type
    return type;
  }
```

看type的初始化
```
//注意这里用了protected关键字，限制了只有子类才能访问
protected TypeToken() {
    this.type = getSuperclassTypeParameter(getClass());
    this.rawType = (Class<? super T>) $Gson$Types.getRawType(type);
    this.hashCode = type.hashCode();
  }
  
  //getSuperclassTypeParameter方法
  //这几句就是上面的说到
  static Type getSuperclassTypeParameter(Class<?> subclass) {
    Type superclass = subclass.getGenericSuperclass();
    if (superclass instanceof Class) {
      throw new RuntimeException("Missing type parameter.");
    }
    ParameterizedType parameterized = (ParameterizedType) superclass;
    //这里注意一下，返回的是Gson自定义的，在$Gson$Types里面定义的TypeImpl等，这个类都是继承Type的。
    return $Gson$Types.canonicalize(parameterized.getActualTypeArguments()[0]);
  }
  
```



## 总结
在了解原理之后，相信大家都知道怎么去获取泛型的类型了。


## 参考资料
https://www.cnblogs.com/doudouxiaoye/p/5688629.html
