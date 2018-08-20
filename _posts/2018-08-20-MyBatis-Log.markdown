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

#### 在mybatis-config.xml中配置的日志打印，具体是怎么实现的？
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

- 读取mybatis-config.xml配置文件
- 创建SqlSessionFactoryBuilder
- - XmlConfigBuilder读取mybatis-config.xml文件中的setting配置,通过持有configuration对象来设置日志的实现
```
	configuration.setLogImpl(resolveClass(props.getProperty("logImpl")));
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

```

- - TypeAliasRegistry
- - 设置LogFactory的具体实现
- 获取Log
- 打印日志