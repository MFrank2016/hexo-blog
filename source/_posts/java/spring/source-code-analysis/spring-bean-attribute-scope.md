---
title: 【Spring源码解读】bean标签中的属性（一）scope 属性
date: 2019-03-05 21:28:43
tags:
- Spring
- 源码解读
categorys:
- Spring
---

## scope 属性说明

在spring中，在xml中定义`bean`时，`scope`属性是用来声明`bean`的作用域的。对于这个属性，你也许已经很熟悉了，`singleton`和`prototype`信手捏来，甚至还能说出`request`、`session`、`global session`，scope不就只有这么几个值吗。

emmm，话不要说太满，容易打脸。常见的各类博客中，一般只会介绍上面说到的几种可能值，但翻一翻官方的说明，你就会发现，事情并没有这么简单。

![](https://i.loli.net/2019/03/07/5c8073af8fd16.png)

这是官方文档中的介绍，scope属性一共有六种可能值，惊不惊喜，意不意外。

![](https://i.loli.net/2019/03/07/5c80742434d0e.png)

下面，就让我们来一一看看各个值代表的意义。

### singleton

`singleton`是scope属性的默认值，当我们把bean的scope属性设置为`singleton`时，代表将对该bean使用单例模式，单例想必大家都熟悉，也就是说每次使用该bean的id从容器中获取该bean的时候，都将会返回同一个bean实例。但这里的单例跟设计模式里的单例还有一些小区别。

设计模式中的单例是通过硬编码，给某个类仅创建一个静态对象，并且只暴露一个接口来获取这个对象实例，因此，设计模式中的单例是相对`ClassLoader`而言的，同一个类加载器下只会有一个实例。

下面就是经典的使用`double-check`实现的懒加载代码：

```java
public class Singleton{
    private static volatile Singleton FRANK;

    public static Singleton getInstance(){
        if (FRANK == null){
            synchronized(this){
                if (FRANK == null) FRANK = new Singleton();
            }
        }
        return FRANK;
    }
}
```

但是在Spring中，`singleton单例`指的是每次从同一个IOC容器中返回同一个bean对象，单例的有效范围是IOC容器，而不是`ClassLoader`。IOC容器会将这个bean实例缓存起来，以供后续使用。

![20190307095059.png](https://i.loli.net/2019/03/07/5c8079052f7de.png)

下面做一个小实验验证一下：

先写一个测试类：

```java
public class TestScope {
	@Test
	public void testSingleton(){
		ApplicationContext context = new ClassPathXmlApplicationContext("test-bean.xml");
		TestBean bean = (TestBean) context.getBean("testBean");
		Assert.assertEquals(bean.getNum() , 0);
		bean.add();
		Assert.assertEquals(bean.getNum() , 1);

		TestBean bean1 = (TestBean) context.getBean("testBean");
		Assert.assertEquals(bean1.getNum() , 1);
		bean1.add();
		Assert.assertEquals(bean1.getNum() , 2);

		ApplicationContext context1 = new ClassPathXmlApplicationContext("test-bean.xml");
		TestBean bean2 = (TestBean) context1.getBean("testBean");
		Assert.assertEquals(bean2.getNum() , 0);
		bean2.add();
		Assert.assertEquals(bean2.getNum() , 1);
	}
}
```

```java
public class TestBean {
	private int num;

	public int getNum() {
		return num;
	}

	public void setNum(int num) {
		this.num = num;
	}

	public void add(){
		num++;
	}
}
```

这是相应的配置文件`test-bean.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN 2.0//EN" "http://www.springframework.org/dtd/spring-beans-2.0.dtd">

<beans>
	<bean id="testBean" class="com.frank.spring.bean.scope.TestBean" scope="singleton"/>
</beans>
```

`testBean`的`scope`为`singleton`，而变量`bean`和`bean1`所指向的实例都是从同一个IOC容器中获取的，所以获取的是同一个bean实例，因此分别对`bean`和`bean1`调用add方法后，num的值就会变成2。而`bean2`是从另一个IOC容器中获取的，所以它是一个新的实例，`num`的值便成了初始值0，调用`add`方法后，num的值变成了1。这样也验证了上面所说的`singleton`单例含义，指的是每一个IOC容器中仅存在一个实例。

### prototype

接下来是另一个常用的scope：`prototype`。与`singleton`相反，设置为`prototype`的bean，每次调用容器的`getBean`方法或注入到另一个bean中时，都会返回一个新的实例。

![20190307191454.png](https://i.loli.net/2019/03/07/5c80fd2fc8df9.png)

与其他的`scope`类型不同的是，Spring并不会管理设置为`prototype`的bean的整个生命周期，获取相关bean时，容器会实例化，或者装配相关的`prototype-bean`实例，然后返回给客户端，但不会保存`prototype-bean`的实例。所以，尽管所有的bean对象都会调用配置的初始化方法，但是`prototype-bean`并不会调用其配置的destroy方法。所以清理工作必须由客户端进行。所以，Spring容器对`prototype-bean` 的管理在一定程度上类似于 `new` 操作，对象创建后的事情将全部由客户端处理。

仍旧用一个小栗子来进行测试：

我们将上面的xml文件进行修改：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN 2.0//EN" "http://www.springframework.org/dtd/spring-beans-2.0.dtd">

<beans>
	<bean id="testBean" class="com.frank.spring.bean.scope.TestBean" scope="prototype"/>
</beans>
```

```java
@Test
public void testPrototype(){
    ApplicationContext context = new ClassPathXmlApplicationContext("test-bean.xml");
    TestBean bean = (TestBean) context.getBean("testBean");
    Assert.assertEquals(bean.getNum() , 0);
    bean.add();
    Assert.assertEquals(bean.getNum() , 1);

    TestBean bean1 = (TestBean) context.getBean("testBean");
    Assert.assertEquals(bean1.getNum() , 0);
    bean1.add();
    Assert.assertEquals(bean1.getNum() , 1);
}
```

这里两次从同一个IOC容器中获取`testBean`，得到了两个不同的bean实例，这就是`prototype`的作用。

接着，我们配置一个初始化方法和销毁方法，来测试一下：

给TestBean类加两个方法：

```java
public class TestBean {
	private int num;

	public void init(){
		System.out.println("init TestBean");
	}

	public void destroy(){
		System.out.println("destroy TestBean");
	}

	public int getNum() {
		return num;
	}

	public void setNum(int num) {
		this.num = num;
	}

	public void add(){
		num++;
	}
}
```

然后在配置文件里设置它的初始化方法和销毁方法：

```xml
<beans>
	<bean id="testBean" class="com.frank.spring.bean.scope.TestBean" scope="prototype"  init-method="init"  destroy-method="destroy"/>
</beans>
```

还是用之前的测试方法：

```java
@Test
public void testPrototype(){
    ApplicationContext context = new ClassPathXmlApplicationContext("test-bean.xml");
    TestBean bean = (TestBean) context.getBean("testBean");
    Assert.assertEquals(bean.getNum() , 0);
    bean.add();
    Assert.assertEquals(bean.getNum() , 1);

    TestBean bean1 = (TestBean) context.getBean("testBean");
    Assert.assertEquals(bean1.getNum() , 0);
    bean1.add();
    Assert.assertEquals(bean1.getNum() , 1);
}
```

输出如下：

```
init TestBean
init TestBean
```

可以看到，仅仅输出了初始化方法`init`中的内容，而没有输出销毁方法`destroy`中的内容，所以，对于`prototype-bean`而言，在xml中配置`destroy-method`属性是没有意义的，容器在创建这个bean实例后就抛弃它了，如果它持有的资源需要释放，则需要客户端进行手动释放才行。这大概就是亲生和领养的区别吧。

![](https://i.loli.net/2019/03/07/5c81044d154bf.png)

另外，如果将一个`prototype-bean`注入到一个`singleton-bean`中，那么每次从容器中获取的`singleton-bean`对应`prototype-bean`都是同一个，因为依赖注入仅会进行一次。

## Request && Session && Application && WebSocket Scopes

![](https://i.loli.net/2019/03/07/5c8106f6ae617.png)

`request` 和 `session` 这两个你也许有所耳闻，但是 `application` 和 `websocket` 是什么鬼？竟然还有这样的神仙scope？？莫方，让我们来一探究竟。

这几个类型的scope都只能在web环境下使用，如果使用 `ClassPathXmlApplicationContext` 来加载使用了该属性的bean，那么就会抛出异常。就像这样：

```java
java.lang.IllegalStateException: No Scope registered for scope name 'request'
```

下面让我们依次来看看这几个值的作用。

### request

如果将scope属性设置为 `request` 代表该bean的作用域为单个请求，请求结束，则bean将被销毁，第二次请求将会创建一个新的bean实例，让我们来验证一下。方便起见，创建一个springboot应用，然后创建一个配置类并指定其扫描的xml：

```java
@Configuration
@ImportResource(locations = {"classpath:application-bean.xml"})
public class WebConfiguration {
}
```

以下是xml中的内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans-3.0.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">
    <bean id="testBean" class="com.frank.springboothello.model.TestBean" scope="request" >
        <aop:scoped-proxy/>
    </bean>
</beans>
```

下面是controller的内容：

```java
@RestController
public class HelloController {

    @Autowired
    private TestBean testBean;

    @Autowired
    private TestBean testBean1;

    @GetMapping("/testBean")
    public void testBean(){
        System.out.println("==========request start==========");
        System.out.println(testBean.getNum());
        testBean.add();
        System.out.println(testBean.getNum());
        System.out.println(testBean1.getNum());
        testBean1.add();
        System.out.println(testBean1.getNum());
        System.out.println("==========request end==========");
    }
}
```

这里还是使用之前的TestBean，也许细心的你会发现，这里有一个乱入的家伙：

```xml
<aop:scoped-proxy/>
```

![20190308093050.png](https://i.loli.net/2019/03/08/5c81c5cc0999a.png)

这是个什么东西？？？

这里其实是声明对该bean使用代理模式，这样做的话，容器在注入该bean的时候，将会使用`CGLib动态代理`为它创建一个代理对象，该对象拥有与原Bean相同的public接口并暴露，代理对象每次调用时，会从相应作用域范围内（这里是`request`）获取真正的`TestBean`对象。

那么，为什么要这样做呢？

因为被注入的bean（`testBean`）和目标bean（`HelloController`）的生命周期不一样，而同一个容器内的bean注入只会发生一次，你想想，`HelloController`是`singleton`的，只会实例化一次，如果不使用代理对象，就意味着我们只能将同一个`request-bean`注入到这个`singleton-bean`中，那之后的每次访问，都将调用同一个`testBean`实例，这不是我们想要的结果。我们希望`HelloController`是容器范围内单例的，同时想要一个作用域为 `Http Request` 的`testBean`实例，这时候，代理对象就扮演着不可或缺的角色了。

另外，值得一提的是，如果我们对一个`scope`为`prototype`的bean使用`<aop:scoped-proxy/>`的话，那么每次调用该bean的方法都会创建一个新的实例，关于这一点，大家可以自行验证。

![](https://i.loli.net/2019/03/08/5c8254687adf6.png)

代理方式默认是`CGLib`，并且只有`public`方法会被代理，`private`方法是不会被代理的。如果我们想要使用基于`JDK`的代理来创建代理对象，那么只需要将aop标签中的`proxy-target-class`属性设置为false即可，就像这样：

```xml
<aop:scoped-proxy proxy-target-class="false"/>
```

但有个条件，那就是这个bean必须要实现某个接口。

我们再来跑一下代码验证一下，启动！

![20190308092733.png](https://i.loli.net/2019/03/08/5c81c5066975f.png)


接下来访问几次`http://127.0.0.1:8080/testBean`，输出如下：

```java
==========request start==========
0
1
1
2
==========request end==========
==========request start==========
0
1
1
2
==========request end==========
```

嗯，一切都在掌控范围之内。

### session

跟`request`类似，但它的生命周期更长一些，是在同一次会话范围内有效，也就是说如果不关闭浏览器，不管刷新多少次，都会访问同一个bean。

我们将上面的xml稍作改动：

```xml
<bean id="testBean" class="com.frank.springboothello.model.TestBean" scope="session" >
    <aop:scoped-proxy/>
</bean>
```

再也运行一下，然后在页面刷新几次：

```java
==========request start==========
0
1
1
2
==========request end==========
==========request start==========
2
3
3
4
==========request end==========
==========request start==========
4
5
5
6
==========request end==========
```

可以看到，num的值一直的增加，可见我们访问的是同一个bean实例。

然后，我们使用另一个浏览器继续访问该页面：

```java
==========request start==========
0
1
1
2
==========request end==========
==========request start==========
2
3
3
4
==========request end==========
```

发现num又从0开始计数了。这样就验证了我们对`session`作用域的想法。

### application

`application`的作用域比`session`又要更广一些，`session`作用域是针对一个 `Http Session`，而a`pplication`作用域，则是针对一个 `ServletContext` ，有点类似 `singleton`，但是`singleton`代表的是每个IOC容器中仅有一个实例，而同一个web应用中，是可能会有多个IOC容器的，但一个Web应用只会有一个 `ServletContext`，所以 `application` 才是web应用中货真价实的单例模式。

![](https://i.loli.net/2019/03/08/5c8215620925e.png)

来测试一下，继续修改上面的xml文件：

```xml
<bean id="testBean" class="com.frank.springboothello.model.TestBean" scope="application" >
    <aop:scoped-proxy/>
</bean>
```

然后再次启动后，疯狂访问。

```java
==========request start==========
0
1
1
2
==========request end==========
==========request start==========
2
3
3
4
==========request end==========
==========request start==========
4
5
5
6
==========request end==========
```

换个浏览器继续访问：

```java
==========request start==========
6
7
7
8
==========request end==========
==========request start==========
8
9
9
10
==========request end==========
```

嗯，验证完毕。

![](https://i.loli.net/2019/03/08/5c82552a6af48.png)

### websocket

`websocket` 的作用范围是 `WebSocket` ，即在整个 `WebSocket` 中有效。

emmmm，说实话，这个验证起来有点麻烦，摸索了半天没有找到正确姿势，所以。。。。如果有知道如何验证这一点的小伙伴欢迎留言补充。

![](https://i.loli.net/2019/03/08/5c824b40b64cd.png)

### global session

也许你会发现，很多博客中说的 `global session` 怎么不见了？？

这你就不知道了吧，因为在最新版本（5.2.0.BUILD-SNAPSHOT）中`global session`早就被移除了。

所以以后再有人问你，scope属性有哪几种可能值，分别代表什么含义的时候，就可以理直气壮的把这篇文章甩他脸上了。

![20190308191406.png](https://i.loli.net/2019/03/08/5c824e7fcae16.png)

## 总结

关于 scope 的介绍到此就告一段落了，来做一个小结：

1. singleton：单例模式，每次获取都返回同一个实例，相对于同一个IOC容器而言。
2. prototype：原型模式，每次获取返回不同实例，创建后的生命周期不再由IOC容器管理。
3. request：作用域为同一个 Http Request。
4. session：作用域为同一个 Http Session。
5. application：作用域为同一个WEB容器，可以看做Web应用中的单例模式。
6. websocket：作用域为同一个WebSocket应用。

希望这篇文章能对你有帮助，如果觉得还不错的话，记得分享给身边的小伙伴哦。

让我们红尘作伴，活得潇潇洒洒。

![](https://i.loli.net/2019/03/08/5c8255c88946b.png)