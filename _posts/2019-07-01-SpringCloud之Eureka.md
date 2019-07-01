---
layout:     post
title:      SpringCloud之Eureka
subtitle:   eureka
date:       2019-07-01
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - redis
---

SpringCloud之Eureka

Eureka简介
========

### 什么是Eureka?

> Eureka是一种基于rest提供服务注册和发现的产品：
> 
> *   **Eureka-Server**: 用于定位服务，以实现中间层服务器的负载平衡和故障转移。
> *   **Eureka-client**：用于服务间的交互，内置负载均衡器，可以进行基本的循环负载均衡

### 为什么使用Eureka

*   提供了完整的服务注册与服务发现，并且也经受住了Netflix的考验，通过注解或简单配置即可
*   与SpringCloud无缝集成，提供了一套完整的解决方案，使用非常方便

### 特性

Eureka 是一种客户端服务发现模式，提供Server和Client两个组件。Eureka Server作为服务注册表的角色，提供REST API管理服务实例的注册和查询。POST请求用于服务注册，PUT请求用于实现心跳机制，DELETE请求服务注册表移除实例信息，GET请求查询服务注册表来获取所有的可用实例。Eureka Client是Java实现的Eureka客户端，除了方便集成外，还提供了比较简单的Round-Robin Balance。配合使用Netflix Ribbon ，可以实现更复杂的基于流量、资源占用情况、请求失败等因素的Banlance策略，为系统提供更为可靠的弹性保证。

eureka的**server，client是相对于注册发现服务的，并不是常见RPC请求的client，server**，服务注册在Eureka Server上，每30秒发送心跳来维持注册状态。客户端90s内都没有发心跳，Eureka Server就会认为服务不可用并将其从服务注册表移除。服务的注册和更新信息会同步到Eureka集群的其他节点。所有zone的Eureka Client每30秒查询一次当前zone的服务注册表获取所有可用服务，然后采用合适的Balance策略向某一个服务实例发起请求。

Eureka是一个AP的系统，具备高可用性和分区容错性。每个Eureka Client本地都有一份它最新获取到的服务注册表的缓存信息，即使所有的Eureka Server都挂掉了，依然可以根据本地缓存的服务信息正常工作。Eureka Server没有基于quorum 机制实现，而是采用P2P的去中心化结构，这样相比于zookeeper，集群不需要保证至少几台Server存活才能正常工作，增强了可用性。但是这种结构注定了Eureka不可能有zookeeper那样的一致性保证，同时因为Client缓存更新不及时、Server间同步失败等原因，都会导致Client访问不到新注册的服务或者访问到过期的服务。

当Eureka Server节点间某次信息同步失败时，同步失败的操作会在客户端下次心跳发起时再次同步；如果Eureka Server和Eureka Client间有网络分区存在(默认的检测机制是15分钟内低于85%的心跳汇报)，Eureka Server会进入自我保护模式，不再把过期服务从服务注册表移除(这种情况下客户端有可能获取已经停止的服务，配合使用Hystrix通过熔断机制来容错和降级，弥补基于客户端服务发现的时效性的缺点)。更复杂的情况见在Eureka官网文档上有详细说明:[Understanding Eureka Peer to Peer Communication](https://github.com/Netflix/eureka/wiki/Understanding-Eureka-Peer-to-Peer-Communication "Understanding Eureka Peer to Peer Communication")

### 同类型的技术还有哪些？

**dubbo**,**Nacos**；**Consul**

### 搭建EurekaServer

现在搭建eureka实在是方便，我这边使用的是IDEA新建的；

*   新建model，选择**spring Initializr** ![选择spring initializr](http://upload-images.jianshu.io/upload_images/10579780-369a07a14d6da226.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   选择**Eureka Server**
    
    ![选择Eureka Server](http://upload-images.jianshu.io/upload_images/10579780-9530a9c4b791f409.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
*   POM文件：
    
        <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>2.1.0.RELEASE</version>
            <relativePath/> <!-- lookup parent from repository -->
        </parent>
        
        <properties>
            <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
            <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
            <java.version>1.8</java.version>
            <spring-cloud.version>Greenwich.M1</spring-cloud.version>
        </properties>
        
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
            </dependency>&lt;dependency&gt;
            &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
            &lt;artifactId&gt;spring-boot-starter-test&lt;/artifactId&gt;
            &lt;scope&gt;test&lt;/scope&gt;
        &lt;/dependency&gt;</dependencies>
        
        <dependencyManagement>
            <dependencies>
                <dependency>
                    <groupId>org.springframework.cloud</groupId>
                    <artifactId>spring-cloud-dependencies</artifactId>
                    <version>${spring-cloud.version}</version>
                    <type>pom</type>
                    <scope>import</scope>
                </dependency>
            </dependencies>
        </dependencyManagement>
        
    
*   application配置如下
    
        server.port=8081
        eureka.client.register-with-eureka=false
        eureka.client.fetch-registry=false
        eureka.client.serviceUrl.defaultZone=http://localhost:${server.port}/eureka/
        
    
    默认该服务注册中心也会将自己作为客户端来尝试注册它自己，所以我们需要禁用它的客户端注册行为，只需要在application.properties配置文件中增加以上信息即可；
    
*   进入管理页面
    

![eureka注册中心](http://upload-images.jianshu.io/upload_images/10579780-91c8cd009ec47ed7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时还没有启动provide，所以目前提供方是空的；

### 搭建 provide Server

*   新建的流程与eureka server一致，只是选择组件的时候选择**eureka Discovery**
    
    ![image](http://upload-images.jianshu.io/upload_images/10579780-92838be37930b724.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
*   POM文件
    

    <properties>
            <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
            <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
            <java.version>1.8</java.version>
            <spring-cloud.version>Greenwich.M1</spring-cloud.version>
        </properties>
    
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            </dependency>
    
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
            </dependency>
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.16.16</version>
                <scope>provided</scope>
            </dependency>
        </dependencies>
    
        <dependencyManagement>
            <dependencies>
                <dependency>
                    <groupId>org.springframework.cloud</groupId>
                    <artifactId>spring-cloud-dependencies</artifactId>
                    <version>${spring-cloud.version}</version>
                    <type>pom</type>
                    <scope>import</scope>
                </dependency>
            </dependencies>
        </dependencyManagement>
    

*   appliaction.yml

    server:
      port: 8080 # 服务端口
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:8081/eureka/ # 服务注册中心地址
    spring:
      application:
        name: eureka-provider # 服务名称
    
    

*   提供服务代码

    
    @RestController @Slf4j public class ComputeController {
    
    @Autowired
    private DiscoveryClient discoveryClient;
    @Autowired
    private Registration registration;
    
    @RequestMapping(value = "/add" ,method = RequestMethod.GET)
    public Integer add(@RequestParam Integer a, @RequestParam Integer b) {
        ServiceInstance serviceInstance = this.serviceInstance();
        Integer r = a + b;
        log.info("/add, host:" + serviceInstance.getHost() + ", service_id:" + serviceInstance.getServiceId() + ", result:" + r);
        return r;
    }
    
    /**
     * 获取当前服务的服务实例
     *
     * @return ServiceInstance
     */
    public ServiceInstance serviceInstance() {
        List<ServiceInstance> list = discoveryClient.getInstances(registration.getServiceId());
        if (list != null && list.size() > 0) {
            return list.get(0);
        }
        return null;
    }
    }
    
    

*   注册中心监控
    
    ![服务提供者](http://upload-images.jianshu.io/upload_images/10579780-ee06b8e24cd3c8bd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    

此时可以看到服务提供者注册成功了

### 搭建 Consumer Server

*   创建方式与 provide Server一致
*   POM 文件内容也一致
*   application.yml

    server:
      port: 8082 # 服务端口
    eureka:
      client:
        register-with-eureka: false     #因此处只是消费，不提供服务，所以不需要向eureka server注册
        service-url:
          defaultZone: http://localhost:8081/eureka/ # 服务注册中心地址
    spring:
      application:
        name: eureka-consumer # 服务名称# 服务名称
    

*   消费者代码

    package com.lc.springcloud.eureka.controller;
    
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RequestMethod;
    import org.springframework.web.bind.annotation.RestController;
    import org.springframework.web.client.RestTemplate;
    
    /**
     * Eureka客户端Controller
     * @author LC
     * @date 2018/11/9
     */
    @RestController
    public class ConsumerController {
    
        @Autowired
        private RestTemplate restTemplate;
    
        @RequestMapping(value = "/add", method = RequestMethod.GET)
        public String add() {
            System.out.println(restTemplate.getForEntity("http://EUREKA-PROVIDER/add?a=10&b=20", String.class).getBody());
            return restTemplate.getForEntity("http://EUREKA-PROVIDER/add?a=10&b=20", String.class).getBody();
        }
    }
     
    

至此，spring cloud之eureka搭建完毕

文中示例代码：[eureka 示例](https://github.com/lvchaogit/SpringCloud/tree/master/eureka "eureka 示例")
