---
layout: post
title: "Spring IOC-Bean"
date: 2016-04-23
description: "Spring IOC-Bean"
tag: spring 
---   


### 1.Spring IoC 容器 和Bean概述
        Spring框架实现控制反转(IoC)的原理。IoC也称为依赖注入(DI).它是一个处理对象依赖的过程。和其他一起工作的其他的对象，只通过构造函数参数，工厂方法或属性注入通过参数实例化或通过工厂实例化返回对象再设置属性。
        当Bean建立后，IoC容器再将这些依赖注入进去。这个过程实际上为反转，因此为控制反转。
        org.springframework.beans 和 org.springframework.context包是Spring框架IoC容器的基础。BeanFactory接口提供了一个先进的配置机制能够管理任何类型的对象。
        ApplicationContext是BeanFactory的一个子接口。它增加了更方便的集成Spring的AOP功能、消息资源处理(使用国际化)、事件发布和特定的应用层。如在Web应用层中使用的WebApplicationContext

### 2.容器概述
        org.springframework.context.ApplicationContext接口代表了Spring IoC容器，并负责上面提到的Beans的实例化、配置和装配。容器通过读取元数据获取对象如何实例化、配置和装配的指示。
        配置元数据可以用XML、Java注解或Java代码来描述。它允许你表示组成应用的对象，以及对象间丰富的依赖关系。
        Spring提供了几个开箱即用的ApplicationContext接口的实现。在独立的应用程序中，通常创建ClassPathXmlApplicationContext或FileSystemXMLApplicationContext的实例。
        虽然XML是定义配置元数据的传统格式，但是你你可以指示容器使用Java注解或者代码作为元数据格式

### 3.Bean概述
        一个Spring IoC容器管理了一个或则多个beans.这些Beans通过你提供给容器的配置元数据进行创建。
        在容器内本身，这些bean定义表示为BeanDefinition对象,它包含了如下的元数据
            .包限定的类名：通常是bean定义的实现类。
            .Bean行为配置元素，这些状态指示bean在容器中的行为(范围，生命周期回调函数等待)
            .其他配置设置应用于新创建的对象中设置，例如：连接池中的连接数或连接池的大小限制
        除了bean定义外，它还包含有关如何创建特定bean的信息,ApplicationContext实现还允许由在容器外创建爱你注册现有的对象。
        这是通过访问ApplicationContext的工厂方法，通过getBeanFactory()返回DefaultListableBeanFactory工厂方法的实现。
        DefaultListableBeanFactory支持通过registerSingleton()和registerBeanDefinition()方法进行注册。然而,典型的应用程序的工作仅仅通过元数据定义的Bean定义beans

### 4.自动装配
        Spring容器可以自动装配相互协作bean的关联关系。自动装配的好处：
             1.自动装配可以显著得减少指定属性或者构造器参数的需求。
             2.当对象发生变化时自动装配可以更新配置。比如如果你需要给一个类添加依赖，那么这个依赖就可以被自动满足而不需要你去修改配置。
        自动装配模式
            Mode                         Explanation墨水解释
            no                          不自动装配。Bean的引用必须用ref元素定义。对于较大的部署不建议改变模式的配置。
            byName                           通过属性名称自动装配。Spring会寻找相同名称的bean并将其与属性自动装配。
            byType                       如果容器中存在一个与指定属性类型相同的bean，那么将与该属性自动装配。
            constructor                      与byType类似，不同之处在于它应用于构造参数。如果在容器中没有找到与构造器参数类型一致的bean


### 五、Bean的生命周期和作用域
    Spring Framework 作用域：
        支持五种作用域，其中三种只能用在基于web的Spring ApplicationContext
    内置支持的作用域分列如下:
        singleton 在每个Spring IOC容器中一个Bean定义对应一个对象实例
        prototype 在一个Bean定义对应多个对象实例
        request 它依据某个bean定义创建而成,该作用域在基于web的Spring ApplicationContext情形下有效
        session 在一个HTTP Session中,一个Bean对应一个实例.该作用于仅基于 Web
        global session 在一个全局的HTTP-Session中,一个bean定义对应一个实例。仅在使用portlet
