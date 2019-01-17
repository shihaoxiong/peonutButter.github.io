---
layout:     post                    # 使用的布局（不需要改）
title:      SpringBoot核心类               # 标题 
subtitle:    SpringBoot 部分核心类 #副标题
date:       2019-01-06              # 时间
author:     BY                      # 作者
catalog: true                       # 是否归档
tags:                               #标签
    - Spring Boot
---

# SpringBoot 部分核心类

​    	**介绍几个Spring Boot  中  SpringApplicationRunListener , ApplicationContextInitializer ，*ApplicationListener*  ，CommandLineRunner 。** 

## 1). SpringApplicationRunListener 

 	其实对于我们来说，没有什么必要自己实现一个SpringApplicationRunListener，在SpringBoot中也只是实现了一个org.springframework.boot.context.event.EventPublishingRunListener。用于在SpringBoot 启动时发布不同的应用事件类型。SpringApplication 实例初始化的时候加载了一批ApplicationListener ， 但是run 方法的流程中并没有看到 。是由SpringApplicationRunListener实现类EventPublishingRunListener完成的。

​	如果我们需要自定义一个SpringApplicationRunListener， 要注意构造方法需要两个参数，一个为org.springframework.boot.SpringApplication，另一个为String[] args 参数列表。



```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.SpringApplicationRunListener;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.core.env.ConfigurableEnvironment;

/**
 * @author xsh
 * @date 2019/1/3
 * @since 1.0.0
 */
public class MySpringApplicationRunListener  implements SpringApplicationRunListener {

    private final SpringApplication application;

    private final String[] args;


    public MySpringApplicationRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
    }


    @Override
    public void starting() {
        System.out.println(" MySpringApplicationRunListener   starting ...  ");
    }

    @Override
    public void environmentPrepared(ConfigurableEnvironment environment) {
        System.out.println(" MySpringApplicationRunListener   environmentPrepared ...  ");
    }

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        System.out.println(" MySpringApplicationRunListener   contextPrepared ...  ");
    }

    @Override
    public void contextLoaded(ConfigurableApplicationContext context) {
        System.out.println(" MySpringApplicationRunListener   contextLoaded ...  ");
    }

    @Override
    public void started(ConfigurableApplicationContext context) {
        System.out.println(" MySpringApplicationRunListener   started ...  ");
    }

    @Override
    public void running(ConfigurableApplicationContext context) {
        System.out.println(" MySpringApplicationRunListener   running ...  ");
    }

    @Override
    public void failed(ConfigurableApplicationContext context, Throwable exception) {
        System.out.println(" MySpringApplicationRunListener   failed ...  ");
    }
}
```

![1546495478428](C:\Users\pame\AppData\Local\Temp\1546495478428.png)

![1546495511687](C:\Users\pame\AppData\Local\Temp\1546495511687.png)

执行结果： 

![1546495554384](C:\Users\pame\AppData\Local\Temp\1546495554384.png)

![1546495566158](C:\Users\pame\AppData\Local\Temp\1546495566158.png)

可以看出在SpringBoot启动时，SpringApplicationRunListener 发布了事件通知。

## 2). ApplicationListener

 	ApplicationListener 对于我们并不陌生，是Spring框架监听者模式的实现。我们可以自定义一些事件，以及Listener 。 如下 : 

  事件模型： 

```java
/**
 * 〈事件测试类〉
 *
 * @author xsh
 * @date 2018/10/10
 * @since 1.0.0
 */
public class EventDO {

    private String name ;

    private String type;

    private String  content;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    @Override
    public String toString() {
        return "EventDO{" +
                "name='" + name + '\'' +
                ", type='" + type + '\'' +
                ", content='" + content + '\'' +
                '}';
    }
}
```

事件测试： 

```java
public class SpringEventListenerDemo {

    public static void main(String[] args) {
        EventDO eventDO = new EventDO();
        eventDO.setName("test");
        eventDO.setType("test-type");
        eventDO.setContent("spring cloud hello  world !!! ");
        // 创建spring 容器
        AnnotationConfigApplicationContext  context = new AnnotationConfigApplicationContext();
        // 增加监听器
        context.addApplicationListener(new MyApplicationListener());
        // 上下文启动
        context.refresh();
        // 发布事件
        context.publishEvent(new MyApplicationEvent(eventDO));
    }


    public static class  MyApplicationListener implements ApplicationListener{
        @Override
        public void onApplicationEvent(ApplicationEvent applicationEvent) {
           Object  eventDO   = applicationEvent.getSource();
            System.out.println(eventDO.toString());
        }

    }


    // 创建自定义事件
    public static class  MyApplicationEvent extends ApplicationEvent {

        public MyApplicationEvent(Object source) {
            super(source);
        }

    }

}
```

## 3). ApplicationContextInitializer

​	ApplicationContextInitializer其实也是Spring框架原有的概念，其主要目的是在ConfigurableApplicationContext 类型或子类refresh之前，对ApplicationContext实例做进一步的定制。

```java
/**
 *
 * @author xsh
 * @date 2019/1/3
 * @since 1.0.0
 */
public class MyApplicationContextInitializer  implements ApplicationContextInitializer {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        System.out.println("    MyApplicationContextInitializer    initialize ...  ");
    }

}
```

同样在spring.factories 加入为配置

```properties
org.springframework.context.ApplicationContextInitializer=com.association.test.MyApplicationContextInitializer
```

## 4). CommandLineRunner

​	CommandLineRunner属于SpringBoot特有的扩展接口。 

​	CommandLineRunner	 需要注意的两点 ： 

*  所有CommandLineRunner 执行点是在SpringBoot 应用 ApplicationContext完全初始化开始工作之后（main 方法的最后一步）/
* 只要在于当前ApplicationContext 应用的 CommandLineRunner  都会被加载执行。