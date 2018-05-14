---
layout: post
title: "maven生命周期"
date: 2014-10-01 
description: "maven生命周期的总结"
tag: 分布式缓存 
---   


### 1.生命周期详解
    clean生命周期:
        1.pre-clean执行一些清理前需要完成工作
        2.clean 清理上一次构件生成的文件
        3.post-clean执行一些清理后需要完成的工作
        default生命周期：
            default生命周期定义了真正构建时所需要执行的步奏，它是所有生命周期中最核心的部门，其包含的阶段如下：
            validate:
            initialize：
            generate-sources：
            process-sources:编译项目的资源文件，一般来说，是编译src/main/resources目录的内部进行变量替换等工作后，复制到项目输出的主classpath目录中
            generate-resources：
            process-resources：
            compile 编译项目的主源码，一般来说，是编译src/mainjava目录的内部进行变量替换等工作后，复制到项目输出的主classpath目录中
            process-classes：
            generate-test-sources：
            process-test-sources:
            test-compile:
            prepare-package:
            package:
            pre-integration-test:
            integration-test:
            post-integration-test:
            verify：
            install：将包安装到Maven本地仓库，供本地其他Maven项目使用
            deploy:将最终的包复制到远程仓库；供其他开发人员和Maven项目使用


### 2.site生命周期：
    site生命 周期的目的是建立和发布项目站点，Maven能够给予POM所包含的信息，自动生成一个友好的站点。生命周期如下：
    #pre-site:执行一些在生成项目站点之前需要完成的工作
    #site ：生成项目站点文档
    #post-site:执行一些在生成项目站点之后需要完成的工作
    #post-deploy :将生成的项目站点发布到服务器上
 
### 3.命令行与生命周期
    从命令行执行Maven任务最主要方式就是调用Maven的生命周期阶段。需要注意的是，各个生命周期是相互独立的，而一个生命周期的阶段是有前后依赖有关的。解析其执行的生命周期阶段：
    $mvn clean :该命令调用clean生命周期的clean阶段，实际执行的节点为clean生命周期的pre-clean和clean阶段
    $mvn test:
    $mvn clena install:
    $mvn clean deploy site-deploy: