---
layout:     post
title:      "Mybatis Log日志源码解读"
subtitle:   " \"Mybatis Log日志源码解读\""
date:       2018-08-20 12:00:00
author:     "echola"
header-img: "img/post-bg-2015.jpg"
tags:
    - Mybatis
---

**我们从Mybatis配置文件中的日志配置开始，来看看它到底是怎么实现的**
![image](/img/in-post/Mybatis-log.png)

```
<configuration>

    <settings>
    	<setting name="logImpl" value="NO_LOGGING"/>
    </settings>

</configuration>
```

##### 从测试代码入手
```
  @Test
  public void shouldReadLogImplFromSettings() throws Exception {
    Reader reader = Resources.getResourceAsReader("org/apache/ibatis/logging/mybatis-config.xml");
    new SqlSessionFactoryBuilder().build(reader);
    reader.close(); 
    
    Log log = LogFactory.getLog(Object.class);
    log.debug("Debug message.");
    assertEquals(log.getClass().getName(), NoLoggingImpl.class.getName());
  }
```

- SqlSessionFactoryBuilder读取mybatis-config.xml配置文件的具体过程：
- - XmlConfigBuilder读取mybatis-config.xml文件中的setting配置,通过持有configuration对象来设置日志的实现

```
	configuration.setLogImpl(resolveClass(props.getProperty("logImpl")));
```
在初始化配置过程中，Builder还初始化了以下对象中的数据：

```
 public BaseBuilder(Configuration configuration) {
    this.configuration = configuration;
    this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
    this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
  }
```


- - Configuration在实例化时注册了常用的日志实现类，并且实现了setLogImpl来指定具体的日志实现类

```
	// 注册常用的日志类
   typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
    typeAliasRegistry.registerAlias("COMMONS_LOGGING", JakartaCommonsLoggingImpl.class);
    typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
    typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
    typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
    typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
    typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);
	// 指定具体的日志实现类
	public void setLogImpl(Class<?> logImpl) {
	    if (logImpl != null) {
	      this.logImpl = (Class<? extends Log>) logImpl;
	      LogFactory.useCustomLogging(this.logImpl);
	    }
	  }
    //LogFactory中useCustomLogging的方法：
    public static synchronized void useCustomLogging(Class<? extends Log> clazz) {
        setImplementation(clazz);
  }
```

- - LogFactory获取具体的Log实例：LogFactory持有Log具体实现的顶级接口，通过此接口可以实例化具体的Log实现类。

```
 private static Constructor<? extends Log> logConstructor;
 //设置实现类的方法：
   private static void setImplementation(Class<? extends Log> implClass) {
    try {
      Constructor<? extends Log> candidate = implClass.getConstructor(new Class[] { String.class });
      Log log = candidate.newInstance(new Object[] { LogFactory.class.getName() });
      log.debug("Logging initialized using '" + implClass + "' adapter.");
      //设置logConstructor,一旦设上，表明找到相应的log的jar包了，那后面别的log就不找了。
      logConstructor = candidate;
    } catch (Throwable t) {
      throw new LogException("Error setting Log implementation.  Cause: " + t, t);
    }
  }
  
   //根据传入的类名来构建Log
  public static Log getLog(String logger) {
    try {
      //构造函数，参数必须是一个，为String型，指明logger的名称
      return logConstructor.newInstance(new Object[] { logger });
    } catch (Throwable t) {
      throw new LogException("Error creating logger for logger " + logger + ".  Cause: " + t, t);
    }
  }
```
- - 具体实现：Log4j,slf4j等日志都实现了上一步的Log接口，例如：

```
public class Log4jImpl implements Log 
```
