---
layout: post
title: "Spring AOP"
date: 2016-04-19
description: "Spring AOP"
tag: spring 
---   



 
### 1.Spring-aop知识整理
AOP：面向切面的编程。OOP面向对象编程。OOP面向的是纵向编写、继承、封装、多态是其三大特性，而AOP是面向横向的编程。
面向切面编写(AOP)通过提供另外一种思考程序结构的途径来弥补面向对象编程的不足。
在OOP中模块化的关键单元是类，而在AOP中模块化的单元则是切面。切面能对关注点进行模块化，例如横切多个类型和对象的事务管理。
AOP框架是Spring的一个重要组成部分。但是Spring-IOC容器不是依赖于AOP，这意味着你有权利选择是否使用AOP，AOP作为Spring-IOC容器的一个补充，使它成为一个强大的中间件解决方案。

AOP在Spring Framework中的作用:
.提供声明式企业服务，特别是为了替代EJB声明式服务。最重要的服务是声明性食物管理。
.允许用户实现自定义切面，用AOP来完善OOP的使用。

### 2.AOP概念：
1.切面(Aspect):一个关注点的模块化，这个关注点可能会横切多个对象。事务管理是J2EE应用中一个关于横切关注点的很好的例子。在Spring AOP中，切面可以使用基于模式或者基于@Aspect注解的方式来实现
2.链接点(Joinpoint):在执行过程中某个特定的店，比如某方法调用的时候或者处理异常的时候。在Spring AOP中，一个链接点总是表示一个方法的执行。
3.通知(Advice):在切面的某个特定的连接点上执行的动作。其中包括around,before,after等不同类型的通知
4.切入点(PointCut):匹配链接点的断言。通知和一个切入点表式关联，并在满足这个切入点的链接点上运行。切入点表示如何和链接点匹配是AOP的核心：Spring缺省使用Aspect切入点语法
5.引入(Introduction):用来给一个类型声明额外的方法或属性(也称为链接类型声明)。Spring允许引入新的接口道任何被代理的对象。例如：你可以使用引入来使一个bean实现IsModified接口，以便简化缓存机制。
6.目标对象:被一个或者多个切面所通知的对象。也被称作被通知(adivsed)对象。既然Spring AOP是通过运行时代理实现的，这个对象永远是一个被代理(proxied)对象。
7.AOP代理(AOP Proxy):AOP框架创建的对象，用来实现切面契约。在Spring中，AOP代理可以是JDK动态代理或者CGLIB代理
8.织入(Weaving):把切面链接到其他的应用程序类型或者对象上，并创建一个被通知的对象。这些可以编译(AspectJ编译器)，类加载时和运行运行时完成。Spring和其他纯Java AOP框架一样，在运行时完成织入。

### 3.通知类型
前置通知(Before-advice):在某连接点之前执行的通知，但这个通知不能阻止链接点之前的执行流程、
后置通知(after-returning-advice):在某连接点正常完成后执行的通知；例如，一个方法没有抛出异常，正常返回。
最终通知(after-advice):当某连接点退出的时候执行通知(不论是正常返回还是抛出异常退出)。
环绕通知(Around-Advice):包围一个连接点的通知。如方法调用。这是最轻大的一种通知类型。环绕通知可以在方法调用前后完成自动以的行为。它也会选择是否继续执行连接带你或者直接返回它自己的返回值或者抛出异常来结束执行。
环绕通知是最常用的通知类型。和AspectJ一样，Spring提供所有类型的通知。推荐使用尽可能简单的通知类型来实现需要的功能

### 4.使用ProxyFactoryBean创建AOP代理
在Spring里面创建一个AOP代理的最基本的方法是使用：org.springframework.aop.framework.ProxyFactoryBean.这个类对应的切入点和通知提供了完成的控制能力(包括他们的应用顺序)。就像其他的FactoryBean实现一样，ProxyFactoryBean引入了一个间接层。如果你定义一个名为foo的ProxyFactorybean,引入foo的对象看到的将不是ProxyFactoryBean实例本身,而是一个ProxyFactoryBean实现里getObject()方法创建的对象。这个方法将创建一个AOP代理，它包装了一个目标对象。
ProxyFactoryBean类本身也是一个javaBean，其属性主要有如下用途：
     1.指定你希望代理的目标对象
     2.指定是够使用CGLIB
主要属性还包括：
    ProxyTargetClass:这个属性为true时,目标类本身代理而不是目标类的接口。如果属性值被设置为true，CGLIB代理将会被创建。
    
|术语|中文|描述|
|:-:|:-:|:-:|
|Joinpoint|连接点|指那些被拦截到的点，在Spring中，这些点指方法(Spring只支持方法类型的链接点)
|PointCut|切入点|指需要配置被增加的JointPoint
|Advice|增强/通知|指拦截到JoinPoint后要做的操作，通知分为前置通知、后置通知、异常通知、最终通知、环绕通知等
|Aspect|切面|切入点和通知的结合
|Target|目标对象|需要被代理的对象

