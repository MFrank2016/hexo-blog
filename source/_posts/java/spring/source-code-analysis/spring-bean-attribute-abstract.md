---
title: 【Spring源码解读】bean标签中的属性（二）abstract 属性和 parent 属性
date: 2019-03-14 08:08:55
tags:
- Spring
- 源码解读
categorys:
- Spring
---

## abstract 属性说明

abstract 在java的语义里是代表抽象的意思，用来说明被修饰的类是抽象类。在Spring中bean标签里的 abstract 的含义其实也差不多，表示当前bean是一个抽象的bean，从而不会为其实例化。

声明一个bean，但是又不让实例化？？？

![](https://i.loli.net/2019/03/14/5c89ad2f033c1.png)

莫方，存在即合理，abstract 属性存在必定有其存在的意义，且听我慢慢道来。

## parent 属性说明

在此之前，我们先说一下另一个属性： parent ，顾名思义，就是一个认爸爸的属性，用来表明当前的bean的老爸是谁，这样就能顺利的继承它的遗产。。。emmm，说错了，继承它的属性。就像这样：

```xml
<bean id="abstractBean" class="com.frank.spring.bean.parent.ParentBean" abstract="true">
    <property name="name" value="Frank" />
    <property name="age" value="18" />
</bean>

<bean id="childBean" class="com.frank.spring.bean.parent.ChildBean" parent="abstractBean">
    <property name="name" value="son"/>
    <property name="height" value="180"/>
</bean>
```

这样我们就有了一个父bean和一个子bean，在childBean中，我们只设置了name和height属性，但由于它老爸是abstractBean，所以默认会继承它的age属性的值，也就是18。在子bean中，可以覆盖父bean中的属性，比如这里的name，在childBean中就重新设置了值。来测试一下：

```java
@Test
public void testAbstract() {
    ApplicationContext context = new ClassPathXmlApplicationContext("test-bean.xml");
    ChildBean childBean = (ChildBean) context.getBean("childBean");
    Assert.assertEquals(childBean.getName(), "son");
    Assert.assertEquals(childBean.getAge(), Integer.valueOf(18));
    Assert.assertEquals(childBean.getHeight(), Integer.valueOf(180));
}
```

以下是ParentBean和ChildBean的定义：

```java
public class ParentBean {
	private String name;
	private Integer age;

    //省略getter和setter方法
}

public class ChildBean {
	private String name;
	private Integer age;
	private Integer height;

    //省略getter和setter方法
}
```

所以，这两个类并不一定要有实际的继承关系，可以是两个普通的类。

## abstract 属性的作用

在Spring中， abstract 属性一般是用来声明抽象bean，将一些公共的属性放到一块，这样就能减少重复的代码。所以在标签中，可以这样设置：

```xml
<bean id="abstractBean" class="com.frank.spring.bean.parent.ParentBean" abstract="true">
    <property name="name" value="Frank" />
    <property name="age" value="18" />
</bean>

<bean id="childA" class="com.frank.spring.bean.parent.ChildBean" parent="abstractBean">
    <property name="name" value="GG"/>
    <property name="height" value="180"/>
</bean>

<bean id="childB" class="com.frank.spring.bean.parent.ChildBean" parent="abstractBean">
    <property name="name" value="DD"/>
    <property name="height" value="185"/>
</bean>
```

这样，abstractBean就起到了一个模板的作用，可以减少重复定义，这里只有简单的几个属性，所以可能看起来不是很直观，可以想象一下，如果abstractBean中有一二十个公用属性，是不是可以少写不少代码呢？

