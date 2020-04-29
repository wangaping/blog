---
layout: post
title: "spring cloud 服务认证和鉴权"
subtitle: '微服务'
author: "W"
header-style: text
tags:
  - spring cloud
---

> 整理下之前项目spring cloud 项目的服务认证和鉴权 
>
> gateway+jwt 方式

项目进行认证和鉴权的主要技术手段其实就是通过一系列filte和interceptor

本案例使用spring boot 2.x 版本 和 spring cloud Finchley(F版本) 请引入自己项目时注意 

如果低版本的项目请自行[百度](https://www.baidu.com)  zuul + OAuth2 + JWT 方式进行鉴权

本案例项目代码 [click me](https://gitee.com/wangaping_0/cloud-auth-gateway-jwt.git)

### 项目结构图

![image-20200428204252600](/img/springcloud/gateway_jwt.png)

**auth-client -a**

> 用于测试的服务A

**auth-client -a**

> 用于测试的服务B

**auth-core**

> 包括一些config文件和一些公共的interceptor

**auth-gateway**

> 项目的网关服务 所有请求的入口处

**auth-sever**

> eureka 服务注册

### 接口
 > localhost:9001/auth/token?userName=admin&password=123123&secret=123123123 	```获取token(为了方便测试程序内置了两个用户 admin和spring 其中admin所有服务均可访问 spring 则只能调用 auth-client-a 服务 请自行测试)```
 >
 > localhost:9001/auth-cloud-client-a/test-a?str=1231 
 >
 > localhost:9001/auth-cloud-client-a/test-a-b?str=1231    ```内嵌调用 auth-client-b的接口  可以测试服务与服务之间的认证```

### 测试

> 测试时请务必先启动auth-server服务！
>
> **所有的请求入口都是gateway服务！**
>
> 测试使用工具PostMan

项目启动成功后 首先访问gateway服务的获取token接口 

  ![image-20200428213814821](/img/springcloud/access_token.png)


​    接下来的每次请求都需要在head中设置 Authorization 携带接口返回的token！

​    接下来测试访问 auth-client-a 服务  **/auth-cloud-client-a/test-a**请求



* 成功情况
  
  ![image-20200428214452433](/img/springcloud/access_client_a_success.png)
  
  > 注意图中标红的地方(新增了一个header属性:Authorization  填入值就是访问/auth/token返回的一串字符) **其次还需要注意的是接口的访问地址是  服务名/请求路径**  可以看到图中正常返回了接口值


* 失败情况

  ![image-20200428215058347](/img/springcloud/access_client_a_failed.png)
  
  > 如果不携带Authorization 则会进行报错 访问就无法进行(在正常项目中需要返回特定的异常码 请自行修改 代码仅做demo使用)
  
   
  
  测试就到此,如果需要复杂测试请自行编写接口进行测试！