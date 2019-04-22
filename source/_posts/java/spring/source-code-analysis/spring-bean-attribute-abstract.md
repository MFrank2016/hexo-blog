---
title: 【Spring源码解读】bean标签中的属性（二）abstract 属性和 parent 属性
date: 2019-04-22 21:08:55
tags:
- Spring
- 源码解读
categorys:
- Spring
---

## abstract 属性说明

`abstract` 在java的语义里是代表抽象的意思，用来说明被修饰的类是抽象类。在Spring中bean标签里的 `abstract` 的含义其实也差不多，表示当前bean是一个抽象的bean，从而不会为它生成实例化对象。

声明一个bean，但是又不让它实例化？？？

![](https://i.loli.net/2019/03/14/5c89ad2f033c1.png)

莫方，存在即合理，`abstract` 属性存在必定有其存在的意义，且听我慢慢道来。

## parent 属性说明

在此之前，我们先说一下另一个属性： `parent` ，顾名思义，就是一个认爸爸的属性，用来表明当前的bean的老爸是谁，这样就能顺利的继承它的遗产。。。emmm，说错了，继承它的属性。就像这样：

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

这样我们就有了一个父bean和一个子bean，在`childBean`中，我们只设置了name和height属性，但由于在xml文件中，通过`parent`属性给它安排了一个老爸是`abstractBean`，所以默认会继承它的age属性的值，也就是18。在子bean中，可以覆盖父bean中的属性，比如这里的name，在childBean中就重新设置了值。

![](https://i.loli.net/2019/04/22/5cbdbe8bb22cd.png)

来测试一下：

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

聪明的你一定发现了，这两个类并不一定要有实际的继承关系，可以是两个普通的类。甚至，`parent`属性所指向的bean可以不用设置某个具体的类，而只设置它是属性，就像这样：

```xml
<bean id="abstractParent" abstract="true">
    <property name="name" value="Frank" />
    <property name="age" value="18" />
</bean>

<bean id="testChild" class="com.frank.spring.bean.parent.ChildBean" parent="abstractParent">
    <property name="name" value="son"/>
    <property name="height" value="180"/>
</bean>
```

```java
@Test
public void testAbstractParent() {
    ApplicationContext context = new ClassPathXmlApplicationContext("test-bean.xml");
    ChildBean childBean = (ChildBean) context.getBean("testChild");
    Assert.assertEquals(childBean.getName(), "son");
    Assert.assertEquals(childBean.getAge(), Integer.valueOf(18));
    Assert.assertEquals(childBean.getHeight(), Integer.valueOf(180));
}
```

可以看到这次我们设置的`abstractParent`并没有为其指定类名，我们也能将它做为`parent`属性的值。

## abstract 属性的作用

在Spring中， `abstract` 属性一般是用来声明抽象bean，将一些公共的属性放到一块，这样就能减少重复的代码。所以在标签中，可以这样设置：

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

这样，`abstractBean`就起到了一个模板的作用，可以减少重复定义，这里只有简单的几个属性，所以可能看起来不是很直观，可以想象一下，如果abstractBean中有一二十个公用属性，是不是可以少写不少代码呢？

## 总结

`abstract` 和 `parent` 属性可以当做用来减少xml重复代码的方法，可以将一些bean的公共属性抽取出来，放到一个公共的bean中，然后使用上面的栗子进行配置，来让xml文件更加简洁。

值得注意的是，`parent`属性配置的bean之间，并不一定需要有实际的继承关系，更多的是属性的继承。被设置为`parent`的bean甚至可以不用映射到某一个具体的类，而仅仅将其当做属性模板来使用。

文章持续更新中，欢迎关注我的公众号留言交流。

![](https://i.loli.net/2019/03/14/5c8a58ba229ca.png)