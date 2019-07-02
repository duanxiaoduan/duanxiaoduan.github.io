---
layout:     post
title:      Spring Cloud之Eureka注册中心及集群
subtitle:   eureka
date:       2019-07-01
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - redis
---

Spring Cloud之Eureka注册中心及集群
-------------------------------------

**创建项目**

创建的网站[http://start.spring.io/](http://start.spring.io/)

创建两个springboot工程，一个作为注册中心，一个作为测试客户端，注意要导入（eureka-server），创建的界面如下

![](https://oscimg.oschina.net/oscnet/139477edf6b238fa0b3ff6ce38cc5965ebd.jpg)

也可以用IDEA 来创建

![](https://oscimg.oschina.net/oscnet/d5c31159d7de478eaaa03959cb29fe739bc.jpg)

依赖的配置

    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    	<modelVersion>4.0.0</modelVersion>
    
    	<groupId>com.ashuo</groupId>
    	<artifactId>eureka-service</artifactId>
    	<version>0.0.1-SNAPSHOT</version>
    	<packaging>jar</packaging>
    
    	<name>eureka-service</name>
    	<description>Demo project for Spring Boot</description>
    
    	<parent>
    		<groupId>org.springframework.boot</groupId>
    		<artifactId>spring-boot-starter-parent</artifactId>
    		<version>1.5.15.RELEASE</version>
    		<relativePath/> <!-- lookup parent from repository -->
    	</parent>
    
    	<properties>
    		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    		<java.version>1.8</java.version>
    		<spring-cloud.version>Edgware.SR4</spring-cloud.version>
    	</properties>
    
    	<dependencies>
    		<!---->
    		<!--<dependency>-->
    			<!--<groupId>org.springframework.cloud</groupId>-->
    			<!--<artifactId>spring-cloud-starter-config</artifactId>-->
    		<!--</dependency>-->
    		<dependency>
    			<groupId>org.springframework.cloud</groupId>
    			<artifactId>spring-cloud-starter-eureka-server</artifactId>
    		</dependency>
    
    		<dependency>
    			<groupId>org.springframework.boot</groupId>
    			<artifactId>spring-boot-starter-test</artifactId>
    			<scope>test</scope>
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
    
    	<build>
    		<plugins>
    			<plugin>
    				<groupId>org.springframework.boot</groupId>
    				<artifactId>spring-boot-maven-plugin</artifactId>
    			</plugin>
    		</plugins>
    	</build>
    
    
    </project>
    

### 在spring boot工程的入口类中加入@EnableEurekaServer，标注好这是注册中心

    @EnableEurekaServer
    @SpringBootApplication
    public class EurekaServiceApplication {
    
    	public static void main(String[] args) {
    		SpringApplication.run(EurekaServiceApplication.class, args);
    	}
    }
    

资源的配置

    spring.application.name=eureka-server
    server.port=3333
    
    eureka.instance.hostname=localhost
    
    #不要向注册中心注册自己
    #eureka.client.register-with-eureka=false
    eureka.client.register-with-eureka=true
    #禁止检索服务
    #eureka.client.fetch-registry=false
    eureka.client.fetch-registry=true
    eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka
    

再新建一个客户端项目，注意依赖配置

    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    	<modelVersion>4.0.0</modelVersion>
    
    	<groupId>com.ashuo</groupId>
    	<artifactId>eureka-client</artifactId>
    	<version>0.0.1-SNAPSHOT</version>
    	<packaging>jar</packaging>
    
    	<name>eureka-client</name>
    	<description>Demo project for Spring Boot</description>
    
    	<parent>
    		<groupId>org.springframework.boot</groupId>
    		<artifactId>spring-boot-starter-parent</artifactId>
    		<version>1.5.15.RELEASE</version>
    		<relativePath/> <!-- lookup parent from repository -->
    	</parent>
    
    	<properties>
    		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    		<java.version>1.8</java.version>
    		<spring-cloud.version>Edgware.SR4</spring-cloud.version>
    	</properties>
    
    	<dependencies>
    		<dependency>
    			<groupId>org.springframework.cloud</groupId>
    			<artifactId>spring-cloud-starter-eureka</artifactId>
    		</dependency>
    
    		<dependency>
    			<groupId>org.springframework.boot</groupId>
    			<artifactId>spring-boot-starter-test</artifactId>
    			<scope>test</scope>
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
    
    	<build>
    		<plugins>
    			<plugin>
    				<groupId>org.springframework.boot</groupId>
    				<artifactId>spring-boot-maven-plugin</artifactId>
    			</plugin>
    		</plugins>
    	</build>
    
    
    </project>
    

    server.port=9001
    spring.application.name=eureka-client
    
    eureka.client.service-url.defaultZone=http://localhost:3333/eureka

在spring boot工程的入口类中加入@EnableDiscoveryClient

    
    @EnableDiscoveryClient
    @SpringBootApplication
    public class EurekaClientApplication {
    
    	public static void main(String[] args) {
    		SpringApplication.run(EurekaClientApplication.class, args);
    	}
    }

写一个小的测试类

    @RestController
    public class TestRestful {
    
        private final Logger logger= Logger.getLogger(getClass());
    
        @RequestMapping(value = "/hello",method = RequestMethod.GET)
        public String home() {
            logger.info("进入home");
            return "hello eureka";
        }
    }

接下来就来验证下

![](https://oscimg.oschina.net/oscnet/bd09c849225d34606b37c87768d78304c42.jpg)

![](https://oscimg.oschina.net/oscnet/e33e94f90e130f15303ca6de269acf0645d.jpg)

这个一个简单的eureka 的demo就成功了，下面我们再来看看集群的方式

在前面的eureka-server上增加两个配置文件 application-peer1.properties ，application-peer2.properties

    application-peer1.properties
    
    spring.application.name=eureka-server
    server.port=1111
    
    eureka.instance.hostname=peer1
    eureka.client.register-with-eureka=false
    eureka.client.fetch-registry=false
    
    eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka
    #eureka.client.service-url.defaultZone=http://peer1:1111/eureka/

    application-peer2.properties
    
    spring.application.name=eureka-server
    server.port=1112
    
    eureka.instance.hostname=peer2
    
    eureka.client.register-with-eureka=false
    eureka.client.fetch-registry=false
    
    #eureka.client.service-url.defaultZone=http://peer1:1111/eureka/
    eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka

再修改下eureka-client 的配置文件

    server.port=9001
    spring.application.name=eureka-client
    
    #eureka.client.service-url.defaultZone=http://localhost:3333/eureka
    
    eureka.client.service-url.defaultZone=http://peer1:1111/eureka,http://peer2:1112/eureka

启动的时候要注意下

![](https://oscimg.oschina.net/oscnet/58a2e8a908245ad2cc444575dcedf233251.jpg)

修改hosts文件，文件地址为C:\\Windows\\System32\\drivers\\etc

    127.0.0.1 peer1
    127.0.0.1 peer2

最后验证集群，然后断掉其中一个，另一个服务"Instances currently registered with Eureka"就会变得有实例

![](https://oscimg.oschina.net/oscnet/e8199d2e1a90773a2d4ab50ab60e3e78350.jpg)

![](https://oscimg.oschina.net/oscnet/ea44b027d2f7b13f41ff4f0de3c6cebd362.jpg)