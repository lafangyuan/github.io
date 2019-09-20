---
layout:     post
title:      "MyBatis系列-MyBatis 源码之Plugins(2)"
subtitle:   " \"MyBatis系列-MyBatis 源码之Plugins(2)\""
date:       2018-09-26 12:00:00
author:     "echola"
header-img: "img/post-bg-2015.jpg"
tags:
    - Mybatis
---

### 可以做什么？
可以拦截Mybatis的核心操作流程并改造。主要包括对以下接口的以下方法：

接口 | 方法 | 描述
---|---|---
Executor|update, query, flushStatements, commit, rollback, getTransaction, close, isClosed|覆盖执行SQL的整个过程，包括组装入参、返回的结果集和执行的SQL过程都可进行拦截
ParameterHandler| getParameterObject, setParameters|拦截SQL的参数组装
ResultSetHandler|handleResultSets, handleOutputParameters|拦截返回的结果集，可以对其重新组装
StatementHandler|prepare, parameterize, batch, update, query|执行SQL的过程，打印SQL，重写SQL等


### 需要注意的事项
插件会覆盖Mybatis的核心执行过程，故需要理解所拦截的接口、方法的具体作用，以免影响整个执行过程的调用。
### 怎么写自己的Plugin?
1.根据自己的需求拦截相应的接口和方法。具体可见上表。例如拦截Executor.update。查看API/源代码可知:

```
public interface Executor {

  ResultHandler NO_RESULT_HANDLER = null;

  int update(MappedStatement ms, Object parameter) throws SQLException;

  ................    
}
```

所以对应的@Intercepts为：
```
@Intercepts(@Signature(method = "update",type = Executor.class,args = {MappedStatement.class,Object.class}))
public class MapResultInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object obj = invocation.proceed();
        return obj;
    }
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target,this);
    }

    @Override
    public void setProperties(Properties properties) {
    }
}
```
2.声明插件配置：

```
//mybaits-config.xml
    <plugins>
        <plugin interceptor="com.xxx.MapResultInterceptor"></plugin>
    </plugins>
```


### 实现原理是什么？
插件继承的父类Plugin使用了动态代理来产生Plugin所有的实现类。[具体可参考](https://github.com/mybatis/mybatis-3/blob/master/src/main/java/org/apache/ibatis/plugin/Plugin.java)：https://github.com/mybatis/mybatis-3/blob/master/src/main/java/org/apache/ibatis/plugin/Plugin.java  核心代码入下：

```
public class Plugin implements InvocationHandler {
    
  public static Object wrap(Object target, Interceptor interceptor) {
    //取得签名Map
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    //取得要改变行为的类(ParameterHandler|ResultSetHandler|StatementHandler|Executor)
    Class<?> type = target.getClass();
    //取得接口
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    //产生代理
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //看看如何拦截
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      //看哪些方法需要拦截
      if (methods != null && methods.contains(method)) {
        //调用Interceptor.intercept，也即插入了我们自己的逻辑
        return interceptor.intercept(new Invocation(target, method, args));
      }
      //最后还是执行原来逻辑
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
}
```

### 应用场景
#### 场景1：查询的List<Map> 数据字段转为驼峰命名

```
import com.ytkj.aoms.util.StringUtils;
import org.apache.ibatis.executor.resultset.DefaultResultSetHandler;
import org.apache.ibatis.executor.resultset.ResultSetHandler;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;

import java.sql.Statement;
import java.util.*;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Created by lafangyuan on 2019/8/6.
 */
@Intercepts(@Signature(method = "handleResultSets",type = ResultSetHandler.class,args = {Statement.class}))
public class MapResultInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object obj = invocation.proceed();
        if(obj instanceof List){
            return handelList(obj);
        }
        return obj;
    }

    private Object handelList(Object obj) {
        List list = (List) obj;
        List datas = new ArrayList();
        for(Object o :list){
            if(o instanceof Map){
                Map<String,Object> data = new HashMap<>();
                Map<String,Object> map = (Map) o;
                for(String key:map.keySet()){
                    data.put(underlineToCamel(key),map.get(key));
                }
                datas.add(data);
            }
        }
        if(list.size()>0&&datas.size()==0){
            return list;
        }else {
            return datas;
        }

    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target,this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
    public  String underlineToCamel(String param){
        if (param==null||"".equals(param.trim())){
            return "";
        }
        param = param.toLowerCase();
        StringBuilder sb=new StringBuilder(param);
        Matcher mc= Pattern.compile("_").matcher(param);
        int i=0;
        while (mc.find()){
            int position=mc.end()-(i++);
            sb.replace(position-1,position+1,sb.substring(position,position+1).toUpperCase());
        }
        return sb.toString();
    }

}

```
