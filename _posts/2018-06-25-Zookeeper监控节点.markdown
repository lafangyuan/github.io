---
layout:     post
title:      "Zookeeper 监控节点"
subtitle:   " \"Zookeeper 监控节点在线离线\""
date:       2018-06-25 12:00:00
author:     "Hux"
header-img: "img/post-bg-2015.jpg"
tags:
    - SpringCloud
---

###### 场景
需要做一个监控服务，监控终端服务是否在线。
###### 原理  
利用Znode临时节点的创建、删除的特性    
*客户端活跃时，临时节点就是有效的。当客户端与ZooKeeper集合断开连接时，临时节点会自动删除*
![image](https://note.youdao.com/yws/public/resource/1f77183ecf1421a6d62d9e1af531ec88/xmlnote/F6C29C70FA5245B7A8D346CFD05A1AF0/11242)
###### 步骤
- watcher监控端创建一个永久型的Znode,并注册这个node的子节点变更事件。
- service服务端创建临时性子节点
###### 代码
- 依赖包:
- - Netflix/curator,基于Zookeeper的二次封装，提供了可用性更好的API和链式调用，可以方便的监控子节点的变更，事件注册只需要注册一次。4.0版本支持zookeeper3.4.X
- 具体文档地址：http://curator.apache.org

```
	<dependency>
			<groupId>org.apache.curator</groupId>
			<artifactId>curator-recipes</artifactId>
			<version>4.0.0</version>
			<exclusions>
				<exclusion>
					<groupId>org.apache.zookeeper</groupId>
					<artifactId>zookeeper</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
```
- 监控端

```
    //创建客户端连接
   RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client1 = CuratorFrameworkFactory.builder().connectString(hostPort)
                .sessionTimeoutMs(5000)//会话超时时间
                .connectionTimeoutMs(5000)//连接超时时间
                .retryPolicy(retryPolicy)
                .build();
        client1.start();
    
    //创建节点
        try {
            client1.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT)
                    .forPath(pnode);
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        //注册子节点的监听事件
        final PathChildrenCache childrenCache = new PathChildrenCache(client1,pnode,true);
        childrenCache.start();
        childrenCache.getListenable().addListener(new PathChildrenCacheListener() {
            @Override
            public void childEvent(CuratorFramework curatorFramework, PathChildrenCacheEvent event) throws Exception {
                log.info("childEvent:{}", JSON.toJSONString(event));
            }
        });
```
- 服务端（被监控端）

```
 zooKeeper = new ZooKeeper(hostPort,3000,this);
        Stat stat = zooKeeper.exists(znode,false);//创建节点，但并不监控
        if(stat==null){
            zooKeeper.create(znode,null, ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.EPHEMERAL);
        }
```
项目地址：https://github.com/echola2016/watcher  
页面截图：
![image](https://note.youdao.com/yws/public/resource/1f77183ecf1421a6d62d9e1af531ec88/xmlnote/40F08B61E5E344F1A34864E42EEB4B73/11299)
