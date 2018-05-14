---
layout: post
title: "maven pom.xml详解"
date: 2014-10-04
description: "pom.xml详解"
tag: maven 
---   

## Maven仓库
    Maven坐标和依赖，坐标和依赖是任何一个构件在Maven世界中的逻辑表示方式，而构件的物理表示方式是文件，Maven通过仓库来统一管理这些文件.
    MVAEN 仓库分类：
                                        maven仓库
    本地仓库                                                            远程仓库
                                                                                    |
                                                                                    |
                                        中央仓库            私服                                    其他公共库

### 1.本地仓库
    重新指定仓库的位置：    在setting.xml文件：
```
    <settings>
        <localRepository>D:\Maven\repositry</localRepository>
    <settings>
```
    这样本地的仓库就被设置到其他的磁盘。
### 2.中央仓库
    在Pom.xml文件中
```
    <repositories>
        <repository>
            <id>center</id>
            <name>Maven Repository Switchboard</name>
            <url>http://repol.maven.org/maven2</url>
            <layout>default</layout>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    <repositories>
```
### 3.私服
    
    A.节省自己的外网带框
    B.加速Maven构件
    C.部署第三方构件
    D.提高稳定性，增强控制
    E.降低中央仓库的负荷
    远程仓库的认证：
    （无需认证也可以访问，但是由于安全的因素，就需要进行一些认证）
       setting.xml中配置仓库认证信息：
```
    <setting>
        <servers>
            <id>my-proj</id>
            <username>repo-user</username>
            <password>repo-pwd</password>
        <servers>
    <settings>
```
    这个Id将认证信息与仓库配置联系在了一起‘

### 4.镜像
    如果仓库X可以提供仓库Y存储所有内容，那么就可以认为X是Y的一个镜像。setting.xml文件镜像的配置
```
<settings>
    <mirrors>
        <mirror>
            <id>maven.net.cn</id>
            <name>one of the centeral mirrors in china </name>
            //配置为中央仓库的镜像，任何对于中央仓库的请求都会转到该镜像、用户也可以使用同样的方法配置其他仓库的镜像（也可以配置认证).
            <url>http://maven.net.cn/content/groups/public/</url>
            <mirrorOf>central</mirrorOf>
        </mirror?
    </mirrors>
</settings>
```
    结合私服配置镜像：私服在setting中配置:
```
    <settings>
    <mirrors>
        <mirror>
            <id>internal-repository</id>
            <name>internal Repository Manager</name>
            <url>http://192.168.1.100/maven</url>
            <mirrorOf>*</mirrorOf>
        </mirror?
    </mirrors>
</settings>
```
    Maven支持更高级的镜像配置：
```
    // 匹配所有远程仓库
    <mirrorOf>  * </mirrorOf>
    // 匹配所有远程仓库，使用localhost的除外，使用file：//协议的除外。也就是匹配所有不在本机上的远程仓库
    <mirrorOf> external:* </mirrorOf>
    // 匹配仓库repo1和repo2,使用逗号分隔多个远程仓库
    <mirrorOf>  repo1,repo2 </mirrorOf>
    // 匹配所有远程仓库，repo1除外！将仓库中从匹配中排除
由于镜像仓库完全屏蔽了被镜像仓库，当镜像仓库不稳定或者停止服务的时候，Maven任将无法访问被镜像仓库，因而无法下载构件。
    <mirrorOf>  *,! repo1 </mirrorOf>
```
