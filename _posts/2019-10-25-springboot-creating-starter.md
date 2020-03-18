---
layout:     post
title:      "自定义SpringBoot Starter"
subtitle:   "以Spring-Boot-Swagger-Starter为例"
date:       2019-10-25 12:00:00
author:     "JoeBig7"
head-style: text
catalog: true
tags:
    - SpringBoot
    - Swagger
---

## 前言
SpringBoot支持我们自己定义starter，在编写starter之前，我们需要知道如何自定义Auto-configuration，然后在进一步创建Starter。通过本文让我们一步步了解如何创建一个完整的Starter.

## Auto-configured Beans
这是第一个概念，自动配置的Bean。在SpringBoot的底层（spring.factories），提供了很多自动配置的类，实现这样一个类通常是@Configuration标注的一个类。并且通常使用@ConditionalOnClass、@ConditionalOnMissingBean、@ConditionalOnBean等注解进行装配时候的约束, 只有检测到了相关的类或者没有声明自己的@Configuration等条件才会触发自动装配。下面看一下DataSource是如何创建Auto Configuration Bean的：
```
@Configuration
@ConditionalOnClass({DataSource.class, EmbeddedDatabaseType.class})
@EnableConfigurationProperties({DataSourceProperties.class})
@Import({DataSourcePoolMetadataProvidersConfiguration.class, DataSourceInitializationConfiguration.class})
public class DataSourceAutoConfiguration {
        ...
}
```
------------------------------------

## 定位自动装配的候选者
上面定义了自动装配的类，但是SpringBoot并不能直接识别出这些候选者。在SpringBoot中个，需要将自动装配的类配置在==META-INF/spring.factories==中，并且需要以**org.springframework.boot.autoconfigure.EnableAutoConfiguration**作为key，**候选类**作为value,具体定义如下所示:
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```
**注意:** 为了防止自动装配的这些类通过component扫描寻找额外的组件，用@Import进行组件的引用。

### 三个关于装配顺序的注解
- @AutoConfigureAfter：可以用来在某个具体的装配类之后进行装配
- @AutoConfigureBefore:可以用来在某个具体的装配类之前进行装配
- @AutoConfigureOrder：和@Order作用类似，可以指定任意的加载顺序。但是该注解只试用于自动装配类


------------------------------------

## Condition注解
SpringBoot提供了很多的@Conditional注解可以在@Configuration类或者@Bean方法中使用。下面一一讲解：

### Class Condition（类条件注解）
通过Class进行条件判断主要有:
- @ConditionalOnClass ： 如果指定的类存在就满足条件
- @ConditionalOnMissingClass ： 如果指定的类不存在就满足条件

代码样例如下：可以看见@ConditionalOnClass声明了如果CustomService类存在就进行装配，@ConditionalOnMissingBean声明了如果CustomService类不存在就进行装配
```
@Configuration
public class MyAutoConfiguration {

	@Configuration
	@ConditionalOnClass(CustomService.class)
	static class CustomConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public CustomService embeddedAcmeService() { ... }

	}

}
```

> 需要注意的是，当注解用在method上，那么JVM会首先进行指定类的加载和引用，而作用在类上则不会有这么操作。


### Bean Conditions（Bean条件注解）
通过Bean进行条件判断主要有:
- @ConditionalOnBean : 如果存在指定的Bean就符合条件
- @ConditionalOnMissingBean ：如果不存在指定的Bean就符合条件

代码样例如下：这边声明了如果不存在myService这是bean那么就会被创建
```
@Configuration
public class MyAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean
	public MyService myService() { ... }

}
```

> 需要注意bean定义的顺序，因为基于bean的判断会根据已经加载到的结果进行判断，所以最好用在自动装配类，可以保证用户定义的bean已经被装配。另外上述的两个注解都不会阻止@Configuration类的创建，但是作用在类上如果不符合要求的不会被注册到容器中。


### Property Conditions（属性条件注解）
通过Property进行条件判断主要有:
- @ConditionalOnProperty： 可以通过指定属性和environment中是否匹配进行条件判断。

> havingValue可以用来指定是否有期望的值，matchIfMissing如果不设置值返回设置

###  Resource Conditions
通过Resource进行条件判断主要有:

- @ConditionalOnResource :指定资源存在则满足条件


###  Web Application Conditions
用来判断是否是Web环境：
- @ConditionalOnWebApplication 


### SpEL Expression Conditions
通过SpEL表达式进行条件判断：
- @ConditionalOnExpression


----------------

## 创建自己的Starter
一个完整的Spring Boot Starter库包含如下组件：
- autoconfigure模块：包含了自动装配的代码
- starter模块：提供对autoconfigure模块的依赖，以及任何需要的额外的依赖。使用starter模块就可以使用完整的功能

### Swagger Starter的创建
接下来，让我们看看如何一步步地创建一个完整的==swagger starter==

### 命名
针对创建的项目名，比如这边创建的是swagger项目,那么就命名auto-configure module为==swagger-spring-boot-autoconfigure==,starter module就命名为==demo-spring-boot-starter==


### 预配置属性
如果想要提供starter的配置属性，需要指定的命名空间，这边是==swagger==。具体的代码如下：
```
@ConfigurationProperties(prefix = "swagger")
@EnableSwagger2
public class SwaggerProperties {
    /**
     * swagger scan package
     */
    private String basePackage;

    /**
     * swagger document title
     */
    private String title = "API";

    /**
     * swagger document description
     */
    private String description;

    /**
     * swagger document access link
     */
    private String url;

    /**
     * swagger document's contact
     */
    private String contact = "JoeBig7";

    /**
     * swagger version
     */
    private String version = "1.0";

    setter/getter...
}
```

### 属性条件筛选
对于swagger的basePackage不能为空，这边使用Condition来判断属性的值。
```
public class OnSwaggerCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String basePackage = context.getEnvironment().getProperty("swagger.basePackage");
        if (Objects.isNull(basePackage)) {
            throw new RuntimeException("please config basePackage first");
        } else {
            return true;
        }
    }
}
```
同时指定义SwaggerCondition注解进行标注
```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnSwaggerCondition.class)
public @interface SwaggerCondition {
}
```

### 自动装配具体代码
在所有的准备工作都做好以后，对Swagger具体的装配进行编写，具体的SwaggerAutoConfiguration代码如下：
```
@Configuration
@SwaggerCondition
@ConditionalOnProperty(name = "swagger.enabled", matchIfMissing = true)
@ConditionalOnClass(name = {"javax.servlet.ServletRegistration", "springfox.documentation.spring.web.plugins.Docket"})
@EnableConfigurationProperties(SwaggerProperties.class)
public class SwaggerAutoConfiguration {


    private SwaggerProperties swaggerProperties;

    public SwaggerAutoConfiguration(SwaggerProperties swaggerProperties) {
        this.swaggerProperties = swaggerProperties;
    }

    @ConditionalOnMissingBean(Docket.class)
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage(this.swaggerProperties.getBasePackage()))
                .paths(PathSelectors.any())
                .build();
    }


    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title(this.swaggerProperties.getTitle())
                .description(this.swaggerProperties.getDescription())
                .termsOfServiceUrl(this.swaggerProperties.getUrl())
                .contact(this.swaggerProperties.getContact())
                .version(this.swaggerProperties.getVersion())
                .build();
    }
}
```

### 定义好Starter
Starter是最后定义的目标项目，对所有的依赖进行总结，这边的pom文件如下：
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.mamba</groupId>
    <artifactId>swagger-spring-boot-starter</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>Spring Boot AutoConfiguration :: Swagger Starter</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.mamba</groupId>
            <artifactId>swagger-spring-boot-autoconfigure</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>2.1.7.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```
> 注意：可以将自动装配的代码全都写到starter中，不是强制分成两个项目的。


### 源码
项目源码参见[传送门](https://github.com/Joebig7/swagger-autoconfigure-demo)