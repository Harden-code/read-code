#### SpringBoot

#### Maven打包的启动类Main-Class：

org.springframework.boot.loader.JarLauncher  jar启动类

org.springframework.boot.loader.WarLauncher war启动类

#### 内嵌Web容器相关接口

org.springframework.boot.web.server.AbstractConfigurableWebServerFactory 内嵌容器创建工场抽象类

org.springframework.boot.web.server.WebServer 内嵌容器接口

org.springframework.boot.web.context.WebServerInitializedEvent 容器初始化事件，可以获取webserver对象

ServletWebServerApplicationContext是AbstractApplicationContext子类，另外重写onRefresh方法。该方法会创建web容器

```java
//org.springframework.context.support.AbstractApplicationContext#refresh;触发
//org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#onRefresh
protected void onRefresh() {
  super.onRefresh();
  try {
    createWebServer();
  }
  catch (Throwable ex) {
    throw new ApplicationContextException("Unable to start web server", ex);
  }
}
```

在SpringBoot启动类会根据webApplicationType来判断，应用类型

```java
ApplicationContextFactory DEFAULT = (webApplicationType) -> {
   try {
      switch (webApplicationType) {
      case SERVLET:
          //ServletWebServerApplicationContext子类
         return new AnnotationConfigServletWebServerApplicationContext();
      case REACTIVE:
         return new AnnotationConfigReactiveWebServerApplicationContext();
      default:
         return new AnnotationConfigApplicationContext();
      }
   }
};
```

#### @EnableXxx注解

通过@Import引入配置类，三种使用方式

直接直接填入配置类，该类上必须要打上@Configuration注解

实现ImportSelector接口的类

```
String[] selectImports(AnnotationMetadata importingClassMetadata);//类的全类名
```

实现ImportBeanDefinitionRegistrar接口的类

```java
default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry,
      BeanNameGenerator importBeanNameGenerator) {

   registerBeanDefinitions(importingClassMetadata, registry);
}
```

涉及到的处理类：ConfigurationClassParser 

#### 注解支持

org.springframework.core.annotation.AnnotatedElementUtils 注解工具类

org.springframework.context.annotation.AnnotatedBeanDefinitionReader 注解bean

org.springframework.context.annotation.ClassPathBeanDefinitionScanner 根据路径扫描spring组件并加入容器（ClassPathScanningCandidateComponentProvider子类）

org.springframework.context.annotation.ConfigurationClassParser 注解Configuration解析，可能会有Import其他注解带入另外的Configuration类

org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader 注解处理器

```
Reads a given fully-populated set of ConfigurationClass instances, registering bean
definitions with the given {@link BeanDefinitionRegistry} based on its contents.
```

org.springframework.context.annotation.ConfigurationClassPostProcessor spring生命周期接口，处理@Configuration类种的注解

#### 自动装配

##### 失效自动装配

- 代码配置方法：1.@EnableAutoConfiguration#exclude；2.@EnableAutoConfiguration#excludeName
- 外部化属性配置放：spring.autoconfigure.exclude

##### 实现原理

通过spi加载META-INF/spring.factories

org.springframework.core.io.support.SpringFactoriesLoader

##### 条件注解

SpringBootCondition，Condition //todo

组合注解

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

标注该注解的类是一个spring配置类，扫描该类的包以及子包查找bean加入到容器，spi加载配置类
