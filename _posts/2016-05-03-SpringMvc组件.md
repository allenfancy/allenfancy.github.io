---
layout: post
title: "Spring-MVC组件"
date: 2016-05-03
description: "Spring-MVC组件"
tag: spring 
---   



### 1.HandlerMapping
    简介：根据request找到相应的的处理器Handler和Interceptors；
    
        AbstractHandlerMapping:采用模板模式设计了HandlerMapping实现的整体结构，子类只需要通过模板方法提供一些初始化指或具体的算法即可。
        在Spring MVC中有很多组件都是采用的这种模式----首先使用一个抽象实现采用模板模式进行整体设计,然后子类通过实现模板方法具体完成业务,在Spring MVC源码的过程中尤其重视对组件接口实现的抽象类的分析。
        HandlerMapping的作用是根据request查找Handler和Interceptors。获取Handler的过程通过模板方法getHandlerInternal交给了子类。
        AbstractHandlerMapping中保存了所有的配置的Interceptor。
        AbstractHandlerMapping的创建其实就是初始化这三个
        Interceptor:interceptors,adaptedInterceptors,mappedInterceptors
    
        作用：HandlerMapping是通过getHandler方法来获取处理Handler和拦截器Interceptor的
        AbstractURLHandlerMapping，从名字看来是通过url进行匹配的。此系列大致原理是将url与对应的Handler保存在一个Map中，getHandlerInternal方法中使用url从Map中获取Handler，AbstractURLHandlerMapping中实现了具体用URL从Map中获取Handler的过程，而Map的初始化则交给了具体的子孙类完成。
        
        这里的Map就是定义在AbstractURLHandlerMapping中的handlerMap,另外还单独定义了处理”/“请求的处理请求rootHandler；在buildPathExposingHandler方法中给Handler注册俩个内部拦截器PathExposingHandlerInterceptor和UriTemplateVariablesHandlerInterceptor,这俩个拦截器分别在preHandle中调用了exposePathWithinMapping和exposeUriTemplateVariables方法将相应的内容设置到了request的属性

### 2.HandlerAdapter
     简介：可以理解为使用处理器干活的人，它里面有三个方法，supports(Object handler)判断是否可以使用某个handler;handler方法是用来具体使用Handler干活；getLastModified是获取资源的Last-Modified

### 3.ViewResolver

### 4.RequestToViewNameTranslator

### 5.HandlerExceptionResolver

### 6.MultipartResolver

### 7.LocaleResolver

### 8.ThemeResolver

### 9.FlashMapManager