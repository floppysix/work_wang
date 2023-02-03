[[Spring]]中最重要的两个部分：[[IOC]]、[[AOP]]
# IOC
	Inversion of COntrol 控制反转——把自己创建资源变成由spring为主动创建，在我们需要的时候直接去获取

spring则是需要提供一个[[IOC容器]]用于存储bean组件，spring则是有两种方式来创建[[IOC容器]]，一是通过BeanFactory，二是通过ApplicationContext。而其中BeanFactory则是spring的内部接口，不提供给开发人员使用；ApplicationContext是一个接口，主要是通过它的两个实现类来创建[[IOC容器]]。
![[Pasted image 20220615222658.png]]
|**类型名**|**简介**|
|---|---|
|ClassPathXmlApplicationContext|通过读取类路径下的 XML 格式的配置文件创建 IOC 容器对象|
|FileSystemXmlApplicationContext|通过文件系统路径读取 XML 格式的配置文件创建 IOC 容器对象|


**Bean管理**
- sping创建对象
- spring注入属性

**Bean管理的两种方式**
- 基于xml配置文件方式
- 基于注解的方式

**基于[[xml配置文件]]进行[[Bean管理]]**
**创建对象**
1. 在resource下新建Spring Config的xml文件![[Pasted image 20220615223615.png]]
2.  在xml文件中使用bean标签，实现对象的创建，bean标签中id属性即为该bean的**唯一标识**，class属性即为类的全路径
	```XML
	<bean id="happyComponent" class="com.atguigu.ioc.component.HappyComponent"/>
	```
3. 通过读取xml文件来获取IOC容器（iocContainer）
```java
// 创建 IOC 容器对象，为便于其他实验方法使用声明为成员变量
    private ApplicationContext iocContainer = new ClassPathXmlApplicationContext("applicationContext.xml");
// 从 IOC 容器对象中获取bean，也就是组件对象
    HappyComponent happyComponent = (HappyComponent)iocContainer.getBean("happyComponent");
```
**TIPS**
- 对一个JavaBean来说，无参构造器和属性的getXxx()、setXxx()方法是必须存在的，特别是在框架中
- n 获取bean可以通过指定id或者根据类型获取，u 但是如果指定类型的bean不唯一，会抛出[[异常]]——[[NoUniqueBeanDefinitionException]]，因此在使用xml管理bean时，建议使用指定id来获取

**注入属性**
1. 构造器注入——使用constructor-arg标签![[Pasted image 20220615225541.png]]
	-  index属性：指定参数所在位置的索引（从0开始）
	- name属性：指定属性名
	- Value属性：按照方法参数依次注入
2. Setter注入——使用property标签以及其name和value属性进行赋值
	-  引用外部已声明的bean——使用property标签以及其name和ref属性进行赋值
		```XML
		<Bean id="happyComponent4" class="com.atguigu.ioc.component.HappyComponent">
    <!-- ref 属性：通过 bean 的 id 引用另一个 bean -->
    <property name="happyMachine" ref="happyMachine"/>
</bean>
```



# 时间相关
## 在MySQL中表示时间的类型
![[Pasted image 20220814150141.png]]
## TIMESTAMP和DATETIME比较
- 相同点
两者都可用来表示YYYY-MM-DD HH:MM:SS 类型的日期。
- 不同点：
他们的的存储方式，大小(字节)，表示的范围不同。
TIMESTAMP，它把客户端插入的时间从当前时区转化为UTC（世界标准时间）进行存储。查询时，将其又转化为客户端当前时区进行返回。
DATETIME，不做任何改变，基本上是原样输入和输出。

**总结**：TIMESTAMP和DATETIME 都可用来表示YYYY-MM-DD HH:MM:SS 类型的日期， 除了存储方式和存储范围以及大小不一样，没有太大区别。但对于跨时区的业务，TIMESTAMP更为合适。

# @Autowired VS @Resource

## Spring常见的DI方式

-   **构造器注入**：利用构造方法的参数注入依赖
-   **Setter注入**：调用Setter的方法注入依赖
-   **字段注入**：在字段上使用@Autowired/Resource注解

事实上，他们的基本功能都是通过注解实现**依赖注入**，只不过`@Autowired`是`Spring`定义的，而`@Resource`是`JSR-250`定义的。大致功能基本相同，但是还有一些细节不同：

-   **依赖识别方式**：@Autowired默认是**byType**可以使用@Qualifier指定Name，@Resource**默认ByName**如果**找不到则ByType**
-   **适用对象**：@Autowired可以对**构造器、方法、参数、字段**使用，@Resource只能对**方法、字段**使用
-   **提供方**：@Autowired是**Spring**提供的，@Resource是**JSR-250**提供的

## 各种DI方式的优缺点

参考Spring官方文档，建议了如下的使用场景：

-   **构造器注入**：**强依赖性**（即必须使用此依赖），**不变性**（各依赖不会经常变动）
-   **Setter注入**：**可选**（没有此依赖也可以工作），**可变**（依赖会经常变动）
-   **Field注入**：大多数情况下尽量**少使用**字段注入，一定要使用的话， **@Resource相对@Autowired**对IoC容器的**耦合更低**

## Field注入的缺点

-   **不能像构造器那样注入不可变的对象**
-   **依赖对外部不可见**，外界可以看到构造器和setter，但无法看到私有字段，自然无法了解所需依赖
-   会导致**组件与IoC容器紧耦合**（这是最重要的原因，离开了IoC容器去使用组件，在注入依赖时就会十分困难）
-   导致**单元测试也必须使用IoC容器**，原因同上
-   **依赖过多时不够明显**，比如我需要10个依赖，用构造器注入就会显得庞大，这时候应该考虑一下此组件是不是**违反了单一职责原则**

  

作者：小亮哥Ya  
链接：https://juejin.cn/post/7080441168462348319  
来源：稀土掘金  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。