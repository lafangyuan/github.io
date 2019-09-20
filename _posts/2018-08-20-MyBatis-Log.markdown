---
layout:     post
title:      "MyBatis系列-Mybatis源码之 Log的实现流程（4）"
subtitle:   " \"MyBatis系列-Mybatis源码之 Log的实现流程（4）\""
date:       2018-11-20 12:00:00
author:     "echola"
header-img: "img/post-bg-2015.jpg"
tags:
    - Mybatis
---

#### 本篇文章内容
- Mybatis的日志如何配置，如何加载配置？
- 核心接口和实现类
- 如何实现只打印SQL，不打印结果集？
- 如何实现只打印部分Mapper的SQL？

##### 官方文档：
http://www.mybatis.org/mybatis-3/zh/logging.html
##### 从配置开始

**我们从Mybatis配置文件中的日志配置开始，来看看它到底是怎么实现的**
![image](https://lafangyuan.github.io/img/in-post/Mybatis-log.png)

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


##### 日志打印是如何实现的？
Mybatis在获取执行一条SQL语句的时候，对Connection,Statement,ResultSet,PreparedStatement作了代理，通过代理实现实现了日志打印。
![image](https://lafangyuan.github.io/img/in-post/Mybatis源码之Log-jdbcLog.png)

###### ConnectionLogger
- 描述：获取数据库连接的日志代理类
- 打印哪些方法的SQL：prepareStatement，createStatement，prepareCall，打印select。
- 调用链：*session.selectList*-->*configuration.getMappedStatement获取MappedStatement*-->*BaseExecutor.query*-->*BaseExecutor.getConnection(MappedStatement.statementLog)*
- BaseExecutor.getConnection会根据Log的具体情况返回是否生成ConnectionLogger

```
  protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
      //如果需要打印Connection的日志，返回一个ConnectionLogger(代理模式)
      return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
      return connection;
    }
  }
```
###### PreparedStatementLogger、StatementLogger
- 描述：PreparedStatement，Statement代理类。
- 打印哪些方法的SQL：execute、executeQuery、executeUpdate、addBatch

##### ResultSetLog
- 描述：ResultSet代理类
- 打印哪些方法的SQL: 调用ResultSet.next()时打印结果集，表头。**只有当Log接口的实现类返isTraceEnabled返回true时才打印。因此，可以通过配置Log实现类对应的日志级别来设置是否打印结果集**

```
//log4j为例
log4j.logger.org.mybatis.example=TRACE

```

#####  可否通过调用方法开启/关闭Mybatis的日志打印？

通过Excutor接口发现，执行SQL的MappedStatement都持有一个Log接口，这个接口的具体设置是在其Builder中创建，也就是说只有在Builder的时候设置才生效。而在获取Log的时候直接返回了设置的statementLog，并且MappedStatement的持有的Log为private,也没有提供公共的设置方法，因此，只能通过修改源代码的方式来设置Log

```
public final class MappedStatement {
    //持有的Log接口
    private Log statementLog;
    ......
    public Builder(Configuration configuration, String id, SqlSource sqlSource, SqlCommandType sqlCommandType) {
      mappedStatement.configuration = configuration;
    .  ....
      String logId = id;
      if (configuration.getLogPrefix() != null) {
        logId = configuration.getLogPrefix() + id;
      }
      mappedStatement.statementLog = LogFactory.getLog(logId);
      mappedStatement.lang = configuration.getDefaultScriptingLanguageInstance();
    }
    
    //获取Log时调用：
      public Log getStatementLog() {
         return statementLog;
      }
      //修改为通过LogFactory获取
      public Log getStatementLog() {
           String logId = this.getId();
            if (configuration.getLogPrefix() != null) {
              logId = configuration.getLogPrefix() + id;
            }
            return LogFactory.getLog(logId);
      }
}
```

动态修改Log打印实现:

```
    @RequestMapping("/log/{type}")
    public Result log(@PathVariable("type")String type) {
        if("on".equals(type)){
            LogFactory.useStdOutLogging();
        }else {
            LogFactory.useSlf4jLogging();
        }
        return new Result().success();
    }
```
