---
title: 回炉重造之Spring(一)
cover: banner.jpg
top_img: /img/tags_index.jpg
tags:
  - Spring
abbrlink: a2e9e7bd
date: 2022-12-17 13:44:08
---
### 前言
这篇文章主要记录Spring重新学习的过程，不是手撕源码的版本因此不涉及到具体的三级缓存结构、bean的具体创建过程等深度问题，这些问题放在重新学习完微服务以后再学习，重新学习会涉及到原理的部分。

### Spring概述
Spring框架是一个开源的J2EE应用程序框架，是针对`bean`的生命周期进行管理的`轻量级`容器(container)。Spring有两个核心：IOC和AOP。
IOC：控制反转，即把创建对象的过程交给Spring进行管理
AOP：面向切面，在不修改源代码的情况下进行功能增强

SpringFramework5的6大部分：
1. Spring IOC工厂
2. Spring AOP编程
3. 持久层集成
4. 事务处理
5. MVC框架集成
6. 注解编程

### 设计模式之工厂模式
面向对象设计中，解决特定问题的时候需要一些特殊的解决方案，例如一个遥控器既可以开空调又可以开电视就是观察者模式的案例。狭义上的设计模式主要是GOF四个大师提出的23种设计模式,常用的主要是工厂、适配器、装饰器、代理、模板等等。

普通模式下创建对象是使用new的方式，例如银行ATM柜台对象需要依赖一个按钮对象，如果按钮对象发生了改变（例如改变它的代码），那么ATM对象也会改变，这样的好处是接口的实现类硬编码在程序中使得构造简单，但是不利于代码的维护。

#### 登录案例以及优化(简单工厂实现)
在登录的案例当中，我们存在以下几个Java文件和分别对应的方法：

1. UerDAO：save(User user);querUserByNameAndPassword(String name,String password) ;
2. UserDAOImpl：save(User user);querUserByNameAndPassword(String name,String password) ;
3. UserService：register(User user); login(String name,String password);
4. UserServiceImpl：register(User user); login(String name,String password);


```java
//普通模式创建对象 使用JUNT来测试
@Test
public void test1(){
	UserService userService = new UserServiceImpl();
	//UserService userService = new UserServiceImplNew();
	//采用工厂模式来获取userService,此时test1和UserServiceImpl就没有耦合，耦合集中到了工厂类BeanFactory当中，至少在这里没有了，后续设计工厂。
	//UserService userService = BeanFactory.getUserService UserServiceImplNew();
	userService.login("name","sammie"); //这里会调用UserDAOImpl中的querUserByNameAndPassword
	User user = new User("sammie","123456");
	userService.register(user);
}
//功能上没有问题，但是假如以后出现了UserServiceImplNew，那么第一行的代码就必须做出修改,进而就要重新编译和部署，这并不符合面向对象的开闭原则。

//工厂类
public class BeanFactory{
	public static UserService getUserService(){
		return new UserServiceImpl();
	}
}
```

#### 反射工厂
在上述的案例中，由于采用new的方式，耦合只是从test1转移到了BeanFactory当中，并没有解决真正的问题，而创建对象除了new以外还可以采用反射的方式，但是反射要求传入全限类名，依旧无法解决耦合的问题。但是全限类名可以通过配置文件解决(后期通过注解的方式),下面以Service层的解耦为案例，同理Dao层也可以在工厂类中进行解耦
```java
//applicationContext.properties
//Properties集合存储 Properties文件集合 注意是Map结构 String:String
//Properties [userService = com.sammie.basic.UserServiceImpl]
//Properties.getProperty("userService") 获取value
userService = com.sammie.basic.UserServiceImpl
//userService = com.sammie.basic.UserServiceImplNew
    
//工厂类BeanFactory.java 使用Properties来构建
public class BeanFactory{
	private static Properties env = new Properties();
	//注意 由于Java中资源是IO类型的，因此尽量避免多次IO读写，使用静待代码块的方式来做
	static{
		try{
		 //获得IO输入流
        InputStream inputStream	= BeanFactory.class.getResourceAsStream("/applicationContext.properties ");
        env.load(inputStream);
        inputStream.close();
        //文件操作 封装Properties集合中 k-v
		}catch (IOException e){
			e.printStackTrack();
		}
    }

	public static UserService getUserService(){
		UserService userService = null;
		try{
			Class clazz = Class.forName(env.getProperty("userService"));
			userService = (UserService)clazz.newInstance();
		} catch (ClassNotFoundException e){
			e.printStackTrace();
		} catch (InstantiationException e){
			e.printStackTrace();
		} catch (IllegaAccessException e){
			e.printStackTrace();
		}
	}
}
```

#### 通用工厂设计
在经过了工厂解耦以后，我们设计的工厂类还存在一个问题就是不管是getUserService还是getUserDAO不仅存在大量相同的代码而且每次新增的时候依然要在工厂类新增代码，因此我们希望可以设计一种通用的工厂类来解决这一问题，使得工厂类可以生产出一切我们想要的方法。
```java
/*
	key 小配置文件中的key [userDAO,userService]
*/
public static Object getBean(String key){
	Object ret = null;
	try{
		Class clazz = Class.forName(env.getProperty(key));
		ret = clazz.newInstance();
	}catch(Exception e){
		e.printStackTrace();
	}
}
```

### Spring的核心API

#### ApplicationContext
作用：Spring提供的ApplicationContext工厂，用于对象的创建
好处：解耦合

ApplicationContext接口类型
定义成接口的好处：屏蔽实现的差异
非web环境(main junit)：ClassPathXmlApplicationContext
web环境：XmlWebApplicationContext

ApplicationContext是一个重量级资源,内存占用相对较多，主要是其实现类占用的内存多，因此不会频繁的创建这个对象，一个应用只创建一个工厂对象，但ApplicationContext是做了锁的设置因此是线程安全的。
```xml
//首先要创建applicationContext.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
        <!--
         \id 名字 唯一
         \class 全限定名
         -->
       <bean id="person" class="com.sammie.basic.pojo.Person"/>
       <bean id="person1" class="com.sammie.basic.pojo.Person"/>
</beans>
```
```java
//测试类中使用
@Test
    public void test1(){
        //1.获得Spring工厂 创建出来的对象叫做bean或者叫componet
        ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("/applicationContext.xml");
        //2.通过工厂获取对象
        Person person = (Person) ctx.getBean("person");
       //Person person = ctx.getBean("person",Person.class);
       //Person person = ctx.getBean(Person.class); 这种模式class属性也必须唯一
       //各种常用API  
        String[] beanNames = ctx.getBeanDefinitionNames();//获取所有的bean的name
        String[] beanNamesForType = ctx.getBeanNamesForType(Person.class); //获取所有Person类型的bean
        boolean isBean = ctx.containsBeanDefinition("person1");//判断指定的bean是否存在 只可以用来判断id
        boolean person1 = ctx.containsBean("person");//判断指定的bean是否存在 可以是name
    }
```
配置文件中需要注意的细节：
```xml
//bean的id值是可以不写的 通过getBean(Person.class)获得对象 注意此时Spirng会存在默认id Person#0 这种适用于只用一次bean的类型
<bean class="com.sammie.basic.pojo.Person"/>

//name属性 用来给bena起别名 此时可以通过getBean("p") 来获得对象 
//与id的作用是类似的 因此可以直接不要id而用name 区别是name可以定义多个,逗号分隔 id唯一
//xml中id的命名遵从变量命名要求 name的命名没有要求 发展到今天id也没限制了
<bean id="person2" class="com.sammie.basic.pojo.Person" name="p,p1,p2"/>
```

#### Spring工厂的底层实现原理(简易非源码版)

Spring在帮助我们创建对象的过程是先读取配置文件，然后用class的值来确定全限定类名，最后用id来确定Object的变量名称返回后供我们使用。

注意：在这个过程当中Spring是会调用我们自己的类的无参构造方法！从这个方面来说Spring通过反射创建对象是**等效**于new的方式。同时也因为是反射的原因，构造方法是公开的还是私有的均可以被访问到。

开发过程中，理论上所有的对象都应该由Spring来创建，但是有个例外是entity的实体对象，由于要操作数据库的原因，实体类应该交给持久层框架来创建，例如Mybatis。

### Spring整合日志框架
Spring整合日志框架后可以在控制台中输出日志，输出一系列Spring运行过程中的一些重要信息。Spring早期都是整合commons-logging.jar,从Spring5.x开始，整合的日志框架变成了logback log4j2(注意不是log4j)。

#### Spring5.x整合log4j

```xml
<dependency>
    <groupId>org.slf4j</groupId>
	<artifactId>slf4j-log4j12</artifactId>
	<version>1.7.25</version>
</dependency>
<dependency>
     <groupId>log4j</groupId>
     <artifactId>log4j</artifactId>
     <version>1.2.17</version>
</dependency>
```

```properties
### 配置根
log4j.rootLogger = console

### 日志输出到控制台显示
log4j.appender.console = org.apache.log4j.ConsoleAppender
log4j.appender.console.Target = System.out
log4j.appender.console.layout = org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern = %d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n
```

### 注入

#### 什么是注入
通过Spring工厂及配置文件，为创建对象的成员变量赋值的过程称作注入
#### 为什么需要注入
通常情况下我们会使用对象.属性名的方式来赋值，但是这样并不能通过配置文件来配置，换而言之会有耦合。
#### 如何进行注入

- 类的成员变量提供get set方法
- 配置Spring配置文件
```xml
<bean id="person" class="com.sammie.basic.pojo.Person">
       <property name="age">
               <value>10</value>
       </property>
       <property name="name">
                <value>sammie</value>
       </property>
</bean>
```
- 通过Spring工厂获取Bean

#### Spring注入的原理分析(简易版)
当在配置文件中指定了bean的class后，Spring将绑定我们自己的实体类，通过property标签的name属性来绑定实体类的属性，通过value标签来具体赋值。具体步骤如下：

1. 首先通过bean中的class用反射的方式来创建对象，**等效**于调用了构造方法。
2. 通过识别property中的name来调用对应对象的set方法（如setName）
3. 给set方法传参，参数来自于value标签中的值 ，并启动类型转换

### Set注入详解
在上述的案例当中我们使用的是value的标签来赋值，但很多情况下成员变量的类型可能不是String或者Integer，例如可能是List或者自定义的DAO，这是就无法使用value标签了，因此针对不同的成员变量类型，property标签内部需要嵌套其他标签。Set注入份为JDK内置类型(String,List,int等)和用户自定义类型(userService等)。

#### Set注入JDK内置类型

1. 8种基本数据类型和String
    使用value标签直接赋值
2. 数组类型
    首先使用list标签，然后嵌套value标签
3. Set集合
    首先使用set标签，然后嵌套value标签，但由于set集合本身是无序的，因此有可能遍历的时候不会按照写在xml里的顺序赋值,**另外set集合本身可以不加泛型，所以可以既可以写value标签也可以写ref用户自定义标签或者嵌套set标签**
4. List集合
    数组和List集合都是用list标签因为List底层也是用数组实现的，然后再嵌套value标签，但和Set集合一样的是可以有其他类型，不同的是list内部的有序的而且可以有重复元素
5. Map集合
    首先使用map标签，然后再嵌套entry，entry当中再嵌套key和value标签，注意由于泛型的原因，key当中需要再指定泛型的类型，如果是基本类型就使用value标签，同样的，value标签也可以是其他任意类型。原理是Map中的每个键值对本质上是一个匿名内部类Map.Entry
6. Properties
    Properties本身就是String，String的类型，直接用props标签嵌套prop标签，prop标签的key就是键，标签内部就是值，内部不需要再和map一样嵌套value
```xml
<bean id="person" class="com.sammie.basic.pojo.Person">
               <property name="age">
                       <value>10</value>
               </property>
               <property name="name">
                        <value>sammie</value>
               </property>
               <property name="emails">
                   <list>
                       <value>sammie@qq.com</value>
                       <value>topsun@163.com</value>
                   </list>
               </property>
               <property name="tel" >
                   <set>
                       <value>1788888888</value>
                       <value>1988888888</value>
                   </set>
               </property>
               <property name="addresses">
                   <list>
                       <value>wuhan</value>
                       <value>shanghai</value>
                   </list>
               </property>
               <property name="qqs">
                   <map>
                       <entry>
                           <key><value>sammie</value></key>
                           <value>8255525</value>
                       </entry>
                       <entry>
                           <key><value>topsun</value></key>
                           <value>666666</value>
                       </entry>
                   </map>
               </property>
               <property name="p">
                   <props>
                       <prop key="key1">value1</prop>
                       <prop key="key2">value2</prop>
                   </props>
               </property>
</bean>
```
#### Set注入用户自定义类型
使用之前的案例，UserService和UserDAO,之前是使用自定义工厂的方式来进行创建的，现在直接定义后由Spring来创建。由依赖关系可以确定先配置UserDAO再配置UserService
第一种方式：
- 为成员变量提供get set方法
- 配置文件中进行赋值(注入)
```xml
<bean id="userService" class="com.sammie.basic.UserServiceImpl">
	<property name="userDAO">
		<bean class="com.sammie.basic.UserDAOImpl"/>
	</property>
</bean>
```

第二种方式：

第一种方式存在两个问题，第一个是当其他Service需要用到UserDAO的时候（例如订单Service），需要再在UserDAO这个bean里面再写一次UserDAOImpl，这就造成了冗余。第二个问题是多个Service就会需要多个UserDAO对象，造成内存的浪费。

```xml
<!-- 先创建userDAO然后再创建其他需要用到UserDAO的Service -->
<bean id="userDAO" class="com.sammie.basci.UserDAOImpl"/>
<bean id="userService" class="com.sammie.basci.UserServiceImpl">
    <property name="userDAO">
        <ref bean="userDAO"/>
    </property>
</bean>
<bean id="orderService" class="com.sammie.basci.OrderServiceImpl">
    <property name="userDAO">
        <ref bean="userDAO"/>
    </property>
</bean>
```
#### Set注入的简化写法
通过嵌套标签的方式需要写大量的标签，我们可以使用简化的方式来解决这一问题。
1. 基于属性简化
```xml
<!-- 基本数据类型和String直接用value属性 List等类型只能嵌套标签 -->
<property name="name" value="sammie"/>

<!-- 用户自定义类型可以使用ref属性 -->
<bean id="userService" clss="com.sammie.basic.UserServiceImpl">
	<property name="userDAO" ref="userDAO"/>
</bean>
```
2. 基于p命名空间简化
```xml
<!-- 在IDEA中用alt+enter的快捷键会自动生成xmls:p的命名空间 -->

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
<bean id="person" class="com.sammie.basic.pojo.Person" p:name="sammie" p:age="10"/>
<bean id="userService" class="com.sammie.basci.UserServiceImpl" p:userDAO-ref="userDAO"/>
</beans>
```
### 构造注入
构造注入是Spring调用构造方法，通过配置文件为成员赋值，具体步骤：

1. 提供有参构造方法
2. Spirng配置文件配置

```xml
<!-- constructor-age标签必须对应在有参构造方法中的变量的名称和数量 -->
<bean id="customer" class="com.sammie.basic.pojo.Customer">
        <constructor-arg>
                <value>sammie</value>
        </constructor-arg>
        <constructor-arg>
                <value>20</value>
        </constructor-arg>
</bean>
```
#### 构造方法重载

构造方法本身是可以重载的，主要有三种情况，分别是个数、类型、顺序参数不同。 

1. 参数个数不同时
```xml
<!-- 参数个数不同时，通过控制constructor-arg标签的数量来实现 -->
<bean id="customer" class="com.sammie.basic.pojo.Customer">
        <constructor-arg>
                <value>sammie</value>
        </constructor-arg>
</bean>
```
2. 参数个数相同，类型不同
```xml
<!-- 注意：由于Spring是通过构造标签来调用的有参构造方法，因此如果第二个属性只写一个就会报错 需要用type类型-->
<bean id="customer" class="com.sammie.basic.pojo.Customer">
        <constructor-arg type="int">
                <value>20</value>
        </constructor-arg>
</bean>
```
### 反转控制于依赖注入(概念)

#### 反转控制

- 反转控制(IOC Inverse of Control),反转也称为转移。
- 控制：对于成员变量的控制权
- 反转控制：在原来的模式当中，对成员变量赋值的控制权=代码，这样就有耦合，而Spring中对成员变量赋值的控制权=Spring配置文件+Spring工厂，从而解除了耦合
- 底层实现：工厂设计模式
#### 依赖注入

- 依赖注入(Dependency Injection DI)
- 注入：通过Spring的工厂及配置文件，为对象(bean，组件)的成员变量赋值
- 依赖注入：当一个类需要另一个类时，就意味着依赖，就可以把另一个类作为本类的成员变量，最终通过Spring配置文件进行注入(赋值)
- 依赖注入的好处：由于通过了Spring来进行管理，解除了耦合

### Spring工厂创建复杂对象

#### 什么是复杂对象
简单对象指的是可以直接通过new构造方法来创建的对象，复杂对象指的是不能同new构造方法创建的对象，例如JDBC中的Connection对象，其采用的就是Class.forName注册后用DriverManager来获取。我们希望通过Spring既可以创建简单对象也可以创建复杂对象。

#### Spring工厂创建复杂对象的三个方式

1. FactoryBean接口

   实现FactoryBean接口 分别实现getObject、getObjectType、isSingleton方法
```java
   //以Connection举例 
   public class ConnectionFactoryBean implements FactoryBean<Connection> {
    //用于书写创建复杂对象的代码
    @Override
    public Connection getObject() throws Exception {
        Class.forName("com.mysql.cj.jdbc.Driver");
        Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/springTest?serverTimezone=UTC", "root", "root");
        return connection;
    }
   
    @Override
    public Class<?> getObjectType() {
        return Connection.class;
    }
   
    @Override
    public boolean isSingleton() {
        return FactoryBean.super.isSingleton();
    }
   }
```
   Spring配置文件的配置
```xml
    <!-- 虽然此时配置和正常的Bean没什么区别 但实现了FactoryBean的类 Spring都会直接返回对应getObject方法里的返回类 即本案例中的connection -->
    <bean id="connectionFactoryBean" class="com.sammie.basic.factoryBean.connectionFactoryBean"/>
```

细节：

- 如果就只想要FactoryBean类型的对象 ctx.getBean("&connectionFactoryBean")

- isSingleton方法返回true的时候只会创建一个复杂对象，返回false每一次都会创建新的复杂对象，根据对象的特点来确定返回值，能共用就true，不能就false

- 在MySQL5.7以后的版本会出现SSL的问题，因此在链接后面可以加上useSSL=false来解决

- 依赖注入的体会 (代码改进)：在上述的方案中，驱动名，连接名和用户密码这四个参数应该定义成成员变量，设置get、set方法，然后在配置文件中注入，这样才能解除耦合


```java
  private String driverClassName;
  private String url;
  private String username;
  private String password;
```
```xml
<bean id="connectionFactoryBean" class="com.sammie.basic.factoryBean.ConnectionFactoryBean">
    <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localohost:3306/springTest?useSSL=false&amp;serverTimezone=UTC"/>
    <property name="username" value="root"/>
    <property name="password" value="root"/>
</bean>
```

FactoryBean的实现原理(简易非源码版)

- 为什么Spring规定FactoryBean接口并且实现getObject()?

- ctx.getBean("conn")获得的是复杂对象Connection而不是ConnectionFactoryBean而要加上("&")

Spring内部运行流程
- 通过connectionFactoryBean(id)获得 ConnectionFactoryBean对象，进而通过instanceof判断出是FactoryBean接口的实现类
- Spring按照规定 getObject() --> Connetcion
- 返回Connection 

2. 实例工厂

为什么有了FactoryBean接口以后还需要实例工厂？

- 避免Spring框架的侵入，当离开了Spring框架以后FactoryBean也就不起作用了
- 整合遗留系统，原有的系统可能已经帮我们实现了我们需要的方法，但没有java文件而只有class文件的时候就没办法通过修改源码来实现FactoryBean

开发步骤:

遗留系统的工厂实例
```java
public class ConnectionFactory {
public Connection getConnectionDriver(){
    Connection connection = null;
    try {
        Class.forName("com.mysql.cj.jdbc.Driver");
        connection= DriverManager.getConnection("jdbc:mysql://localhost:3306/springTest?serverTimezone=UTC", "root", "root");
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    } catch (SQLException e) {
        e.printStackTrace();
    }
    return connection;
}
}
```

注册由Spring来进行管理的conn

```xml
<bean id="connectionFactory" class="com.sammie.basic.factoryBean.ConnectionFactory"/>
<bean id="conn" factory-bean="connectionFactory" factory-method="getConnectionDriver"/>
```

测试由Spring帮我们生成的conn
```java
@Test
public void test7(){
     ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
     Connection conn = (Connection) ctx.getBean("conn");
     System.out.println("conn = " + conn);
}
```



3. 静态工厂

静态工厂和实例工厂的区别就是静态方法和实例方法的区别。主要在于配置项有所不同，注意此时的getConnectionDriver已经调整为static

```xml
<bean id="connectionFactory" class="com.sammie.basic.factoryBean.ConnectionFactory" factory-method="getConnectionDriver"/>
```

### 控制Spring工厂创建对象的次数

- 为什么要控制对象的创建次数？

```markdown
好处：节省不必要的内存浪费
```

- 什么样的对象只创建一次？

```markdown
1.SqlSessionFactory
2.DAO
3.Service
```

- 什么样的对象每一次都要创建新的？

```markdown
1.Connection
2.SqlSession | Session
3.Action
```

如何控制简单对象的创建次数
```xml
	<bean id="cat" class="com.sammie.basic.pojo.Cat" scope="singleton|prototype"/> <!-- 默认值是singleton -->
```
```java
	@Test
	public void test8(){
    ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
    Cat cat1 = (Cat) ctx.getBean("cat");
    Cat cat2 = (Cat) ctx.getBean("cat");
    System.out.println("cat1 = " + cat1);
    System.out.println("cat2 = " + cat2);//一样的地址
	}
```
如何控制复杂对象的创建次数
```java
@Override
public boolean isSingleton() {
    return FactoryBean.super.isSingleton();//FactoryBean中默认的isSingleton是true 即只创建一次
}
```
如果没有isSingleton方法，例如我们遗留系统中的方法，那么在注册的时候依然使用scope来指定 

### 对象的生命周期

对象生命周期指的是一个对象创建、存活、消亡的一个完整过程。

#### Spring生命周期的三个阶段

- 创建阶段
创建对象阶段分为singleton和prototype，当创建的对象是singleton时，Spring工厂在创建的时候就会直接创建对象，而创建的对象是prototype时，Spring工厂会在获取对象的同时创建对象，即ctx.getBean("")时，但是如果是prototype类型的对象我们也希望可以在Spring工厂创建的时候就直接创建对象，可以通过指定lazy-init="true"来实现。

- 初始化阶段
  初始化方法由程序员提供而不是Spring提供，初始化方法由Spring工厂来调用。

```markdown
初始化的两种方式：
1.InitializingBean接口 实现afterPropertiesSet方法 缺点:耦合了Spring
2.在对象中定义普通方法 方法名称随便起 由配置文件指定 init-method="方法名"
细节：
如果均实现了接口也定义了普通方法，会先运行接口的实现方法再运行普通方法
注意：Spring在创建完对象以后会先进行注入，再进行初始化
初始化操作主要用来进行资源初始化：IO、数据库、网络等
```

- 销毁对象
  Spring会在工厂关闭之前即ctx.close()的时候销毁对象，销毁方法由程序员定义但和初始化一样也是由Spring调用
```markdown
细节：
销毁的两种方式：
1.DisposableBean 实现destroy方法 缺点：耦合了Spring
2.在对象定义普通方法，在配置文件中指定方法名 destroy-method
细节：
销毁方法只适用于singleton类型的对象
销毁操作主要用来完成资源的释放 IO、数据库等
```

### 配置文件参数化

把Spring配置文件中需要经常修改的字符串信息，转移到一个更小的配置文件中，主要是以数据库参数为代表，这样的好处是可维护的同时利于解耦，不至于让不懂Spring的人修改以后会误碰其他参数

#### 配置文件参数的开发步骤

设置工厂类参数

```java
public class ConnectionFactoryBean implements FactoryBean<Connection> {
    private String driverClassName;
    private String url;
    private String username;
    private String password;
}
```

配置bean 

```xml
<context:property-placeholder location="classpath:db.properties"/>
<bean id="connectionFactoryBean" class="com.sammie.basic.factoryBean.ConnectionFactoryBean">
        <property name="driverClassName" value="${jdbc.driverClassName}"/>
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
</bean>
```

创建转移后的数据库配置文件db.properties

```properties
jdbc.driverClassName = com.mysql.cj.jdbc.Driver
jdbc.url = jdbc:mysql://localhost:3306/springTest?serverTimezone=UTC
jdbc.username = root
jdbc.password = root
```

### 自定义类型转换器

在我们平时配置bean的时候，如果bean的字段类型是int而value只能写字符串，很明显Spring为我们构建了一个类型转换器，这个转换器就是Converter，有时候我们可能需要自定义的类型转换器，比如Date这个类型Spring就是没有提供类型转换的

自定义类型转换器开发步骤：

构建实体类且设置set方法

```java
public class People {
    private Integer id;
    private Date date;
    public void setId(String id) {
    }
    public void setDate(Date date) {
        this.date = date;
    }
    @Override
    public String toString() {
        return "People{" +
                "id=" + id +
                ", date=" + date +
                '}';
    }
}
```

编写自定义类型转换器

```java
public class MyConverter implements Converter<String, Date> {
    /**
     * 把String类型转换成Date类型
     * @param source 日期字符串 由spring来传递值
     * @return 返回值直接由spring来注入date
     */
    @Override
    public Date convert(String source) {
        Date date = null;
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        try {
            date  = sdf.parse(source);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return date;
    }
}
```

配置转换器，且由Spring来注册转换器，注意conversionService和converters是注册器工厂的内置属性，不能指定其他属性

```java
<bean id="myconverter" class="com.sammie.basic.converter.MyConverter"/>
<bean id="conversionService" class="org.springframework.context.support.ConversionServiceFactoryBean">
    <property name="converters">
            <set> 
                   <ref bean="myconverter"/>
            </set>
    </property>
</bean>
<bean id="people" class="com.sammie.basic.pojo.People">
    <property name="id" value="1"/>
    <property name="date" value="2022-12-28"/>
</bean>
```

细节：

- 在MyConverter当中SimpleDateFormat的格式需要指定，应该修改为成员变量并设置set方法，由Spring进行依赖注入
- ConversionServiceFactoryBean的id以及name不可以随便写 因为其内部指定了属性名称
- Spring其实内置了日期类型的转换器，但是其以反斜线的`\`的格式，如果按照反斜线的方式则无需自定义转换器

### 后置处理Bean

BeanPostProcessor(接口)作用：对Spring工厂创建的对象，进行再加工，注意这一章是简单介绍BeanPostProcessor，后面AOP重点介绍 

回顾一下Spring工厂创建对象的流程：

```markdown
1.首先读取配置文件中的bean标签
2.调用bean绑定的class类的构造方法
3.如果存在属性则进行注入
4.如果实现了initializingBean接口则初始化
5.如果存在自定义初始化方法则执行init-method里指定的方法
```

BeanPostProcessor接口的两个实现方法

```markdown
postProcessBeforeInitialization:在initializingBean之前进行加工操作
postProcessAfterInitialization :在自定义初始化方法(init-method)之后加工
```

实际工作中，很少处理Spring的初始化操作，因此大部分时候没必要区分是After还是Before，只需要实现其中一个即可。

开发步骤：

构造实体类

```java
public class Category {
    private String name;
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
}
```

实现BeanPostProcessor接口 根据解耦的思想 将需要修改的参数外置注入

```java
public class MyBeanProcesor implements BeanPostProcessor {
    private String args;
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        Category category = (Category) bean;
        category.setName(args);
        return category;
    }

    public void setArgs(String args) {
        this.args = args;
    }
}
```

Spring的配置文件进行配置

```xml
<bean id="category" class="com.sammie.basic.pojo.Category">
        <property name="name" value="sammie"/>
</bean>
<bean id="mybeanprocessor" class="com.sammie.basic.beanProcessor.MyBeanProcesor">
        <property name="args" value="tom"/>
</bean>
```

细节：

- BeanPostProcessor会对所有由Spring创建的对象进行加工！
- 如果存在多个BeanPostProcessor，则按照在配置文件中的从上到下顺序执行

