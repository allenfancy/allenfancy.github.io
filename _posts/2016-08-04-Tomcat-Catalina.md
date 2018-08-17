---
layout: post
title: "Tomcat-Catainla解析"
date: 2016-08-04
description: "Tomcat-Catainla解析"
tag: TOMCAT
---   

---


### 1. Catalina?

### 2. Digester

### 3. 创建Server
#### 3.1 Server解析
    1. 创建server
	org.apache.catalina.core.StandardServer
	2. 创建Naming资源管理器
	org.apache.catalina.deploy.NamingResourcesImpl
	org.apache.catalina.LifecycleListener
	3. 添加标准的service
	org.apache.catalina.core.StandardService
	org.apache.catalina.LifecycleListener
	4. 标准执行器
	org.apache.catalina.core.StandardThreadExecutor
	5 . 添加连接器
	org.apache.catalina.connector.Connector
	org.apache.catalina.LifecycleListener
	org.apache.coyote.UpgradeProtocol
	6. 添加规则集合
	NamingRuleSet
	EngineRuleSet
	HostRuleSet
	NamingRuleSet

#### 3.2 Engine解析
	org.apache.catalina.core.StandardEngine
	org.apache.catalina.startup.EngineConfig
	org.apache.catalina.Engine
	org.apache.catalina.Cluster
	org.apache.catalina.LifecycleListener
#### 3.3 Host的解析
     
#### 3.4 Context解析

### 4. web应用加载
	web应用加载属于Server启动的核心处理过程。
	Catalina对web应用的加载主要由StandardHost、HostConfig、StandardContext、ContextConfig、StandarWrapper来完成.

### 5. web请求处理

### 6. DefaultServelt和JspServlet
