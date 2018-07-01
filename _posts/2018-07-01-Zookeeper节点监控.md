
---
---
layout: post
title: "Hank Quinlan, Horrible Cop, Launches Site"
date: 2014-04-30
---
- 参考：https://github.com/abel533/Mybatis-Spring/tree/mybatis-oracle
- 通用Mapper文档：https://github.com/abel533/Mapper
### pom.xml配置
```
		<!--mybatis  -->
		<dependency>
			<groupId>tk.mybatis</groupId>
			<artifactId>mapper-spring-boot-starter</artifactId>
			<version>1.1.3</version>
		</dependency>
		<dependency>
			<groupId>com.github.pagehelper</groupId>
			<artifactId>pagehelper-spring-boot-starter</artifactId>
			<version>1.2.1</version>
		</dependency>
		<dependency>
			<groupId>org.mybatis.generator</groupId>
			<artifactId>mybatis-generator-core</artifactId>
			<version>1.3.5</version>
			<scope>compile</scope>
			<optional>true</optional>
		</dependency>
```
### application.propperties配置
```
mapper.mappers=com.ytkj.aoms.util.AomsCommonMapper
mapper.not-empty=true
mapper.identity=ORACLE
#分页
pagehelper.helperDialect=oracle
pagehelper.reasonable=true
pagehelper.supportMethodsArguments=true
pagehelper.params=count=countSql
```
AomsCommonMapper继承了Mapper接口即可:
```
public interface  AomsCommonMapper<T> extends Mapper<T> {
}
```
每个单表对应的Mapper继承AomsCommonMapper即可：
```
public interface GateMapper extends AomsCommonMapper<Gate>{
}
```
#### SpringBoot配置注意事项
- Mapper扫描：  
Application注解添加 **@MapperScan(basePackages = "com.ytkj.aoms.mapper")** 
- 事务控制：  
Application注解添加 **@EnableTransactionManagement**，需要事务的类（Service等）在方法或者类上面添加 **@Transactional**
#### 主要方法使用：
- 根据类查询：
```
	AbnormalReason r = new AbnormalReason();
	r.setNameC("CA");
	List<AbnormalReason> list= abnormalReasonMapper.select(r);
```
- 根据Example查询：
```
       Example example =  new Example(AbnormalReason.class);
		example.createCriteria().andEqualTo("nameE","E")
				.andLike("nameC","%C%");
		List<AbnormalReason> list = abnormalReasonMapper.selectByExample(example);
```
#### 代码生成器
#### ID生成规则
见：https://gitee.com/free/Mapper/blob/master/wiki/mapper3/3.Use.md
- Oracle使用
- - 如果是sequence加触发器： @GeneratedValue(generator = "JDBC")
- - 如果是sequence: @GeneratedValue(strategy = GenerationType.IDENTITY,generator = "select SEQ_task_id.nextval from dual")
#### MyBatis的一级缓存和二级缓存
##### 一级缓存
- 什么情况下有效？在一个事务下面有效，一个sqlSession打开和关闭的范围内，所以，如果没有事务，那么一级缓存是失效的。
- **注意：** 打开两个sqlsession: sqlsession1,sqlsession2,sqlsession1.select从一级缓存中取数据，sqlsession2更新此条数据，sqlsession1.select再次获取数据，还是缓存中的。
##### 二级缓存
- SessionFactory层面给各个SqlSession 对象共享
- 怎么配置：
```
<setting name="cacheEnabled" value="true" /> （或@CacheNamespace） 
<cache/>,<cache-ref/>或@CacheNamespace
```
此外，POJO类必须实现序列化接口。
- 坑：缓存区按照namespace进行划分，一般一个mapper对应一个namespace,开辟一块内存区域来存储mapper查询的结果集。对于一些业务上经常改动的数据，避免使用二级缓存。
- 添加namespace的接口中Select方法默认全部开启二级缓存，缓存刷新时间@CacheNamespace(flushInterval = 10000)，单位为毫秒，如果不设置，则一直有缓存。
###### 问题：
- 通用Mapper中配置setting cacheEnabled到底有没有作用？
- @Option(timeout)作用是什么？
- @CacheNamespace默认开启Mapper的上的select全部缓存？
```
@CacheNamespace
public interface SysUserMapper extends AomsCommonMapper<SysUser>{

    @Select("select sys_resource.* from sys_resource sys_resource \n" +
            "right join \n" +
            "(select sys_role_resource.resource_id from sys_role_resource where sys_role_resource.role_id =\n" +
            "(select sys_user_role.role_id  from sys_user_role\n" +
            "where sys_user_role.user_id = #{userId})) selectedResource on selectedResource.resource_id = sys_resource.id")
    List<SysResource> querySysResourceByUserId(@Param("userId")String userId);

    @Select("select count(*) from sys_user where user_name = #{userName}")
    int queryByUserName(SysUser sysUser);

    @SelectProvider(type = SqlPrivider.class, method = "queryLike")
    List<SysUserVO> queryLike(SysUser sysUser);

    @Options(useCache = true,  timeout = 10000)
    @Select("select * from sys_user where user_name = #{userName}")
    SysUser queryOneByUserName(@Param("userName") String userName);
}
```
##### 约定|坑
- Mybatis默认的BooleanTypeHandle中转换布尔值，0表示false,其他数字表示true
