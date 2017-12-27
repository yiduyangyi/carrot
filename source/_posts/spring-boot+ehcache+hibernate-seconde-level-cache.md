---
title: spring boot集成ehcache 2.x 用于hibernate二级缓存
tags: ["spring boot", "ehcache", "hibernate"]
categories: ["java", "spring boot"]
---


# spring boot集成ehcache 2.x 用于hibernate二级缓存

|      name     |    phone    |
|---------------|-------------|
| *yangjinfeng* | 18521061962 |
| carrot        | 18616798040 |
|               |             |

本文将介绍如何在spring boot中集成ehcache作为hibernate的二级缓存。各个框架版本如下

* spring boot：1.4.3.RELEASE  
* spring framework: 4.3.5.RELEASE  
* hibernate：5.0.1.Final（spring-boot-starter-data-jpa默认依赖）  
* ehcache：2.10.3  

## 项目依赖
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-jpa</artifactId>
	</dependency>

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-cache</artifactId>
	</dependency>

	<dependency>
	    <groupId>org.hibernate</groupId>
	    <artifactId>hibernate-ehcache</artifactId>
	    <exclusions>
	        <exclusion>
	            <groupId>net.sf.ehcache</groupId>
	            <artifactId>ehcache-core</artifactId>
	        </exclusion>
	    </exclusions>
	</dependency>

	<dependency>
	    <groupId>net.sf.ehcache</groupId>
	    <artifactId>ehcache</artifactId>
	    <version>2.10.3</version>
	</dependency>

## Ehcache简介

ehcache是一个纯java的缓存框架，既可以当做一个通用缓存使用，也可以作为将其作为hibernate的二级缓存使用。缓存数据可选择如下三种存储方案

* MemoryStore – On-heap memory used to hold cache elements. This tier is subject to Java garbage collection.

* OffHeapStore – Provides overflow capacity to the MemoryStore. Limited in size only by available RAM. Not subject to Java garbage collection (GC). Available only with Terracotta BigMemory products.

* DiskStore – Backs up in-memory cache elements and provides overflow capacity to the other tiers.

## hibernate二级缓存配置
hibernate的二级缓存支持entity和query层面的缓存，org.hibernate.cache.spi.RegionFactory各类可插拔的缓存提供商与hibernate的集成。  

	# 打开hibernate统计信息
	spring.jpa.properties.hibernate.generate_statistics=true

	# 打开二级缓存
	spring.jpa.properties.hibernate.cache.use_second_level_cache=true

	# 打开查询缓存
	spring.jpa.properties.hibernate.cache.use_query_cache=true

	# 指定缓存provider
	spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.ehcache.SingletonEhCacheRegionFactory

	# 配置shared-cache-mode
	spring.jpa.properties.javax.persistence.sharedCache.mode=ENABLE_SELECTIVE

关于hibernate缓存相关的所有配置可参考[hibernate5.0官方文档#缓存](http://docs.jboss.org/hibernate/orm/5.0/userguide/html_single/Hibernate_User_Guide.html#caching)

## ehcache配置文件
ehcache 2.x配置文件样板参考[官方网站提供的ehcache.xml](http://www.ehcache.org/ehcache.xml)。本例中使用的配置文件如下所示  

	<?xml version="1.0" encoding="UTF-8"?>

	<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	     xsi:noNamespaceSchemaLocation="http://www.ehcache.org/ehcache.xsd"
	     updateCheck="true" monitoring="autodetect"
	     dynamicConfig="true">

		<diskStore path="user.dir/cache"/>
		<transactionManagerLookup class="net.sf.ehcache.transaction.manager.DefaultTransactionManagerLookup"
		                          properties="jndiName=java:/TransactionManager" propertySeparator=";"/>
		<cacheManagerEventListenerFactory class="com.yangyi.base.ehcache.CustomerCacheManagerEventListenerFactory" properties=""/>

		<defaultCache
           maxEntriesLocalHeap="0"
           eternal="false"
           timeToIdleSeconds="1200"
           timeToLiveSeconds="1200">
	      <!--<terracotta/>-->
	    </defaultCache>

	    <cache name="entityCache"
	            maxEntriesLocalHeap="1000"
	            maxEntriesLocalDisk="10000"
	            eternal="false"
	            diskSpoolBufferSizeMB="20"
	            timeToIdleSeconds="10"
	            timeToLiveSeconds="20"
	            memoryStoreEvictionPolicy="LFU"
	            transactionalMode="off">
	        <persistence strategy="localTempSwap"/>
	        <cacheEventListenerFactory class="com.yangyi.base.ehcache.CustomerCacheEventListenerFactory" />
	    </cache>

	    <cache name="org.hibernate.cache.internal.StandardQueryCache"
	           maxEntriesLocalHeap="5" eternal="false" timeToLiveSeconds="120">
	        <persistence strategy="localTempSwap" />
	        <cacheEventListenerFactory class="com.yangyi.base.ehcache.CustomerCacheEventListenerFactory" />
	    </cache>

	    <cache name="org.hibernate.cache.spi.UpdateTimestampsCache"
	           maxEntriesLocalHeap="5000" eternal="true">
	        <persistence strategy="localTempSwap" />
	        <cacheEventListenerFactory class="com.yangyi.base.ehcache.CustomerCacheEventListenerFactory" />
	    </cache>
	</ehcache>

## ehcache事件监听
在ehcache中，有两大类事件，一是cacheManager相关的事件，如cache的init/added等；二是cahche相关的事件，如cache put/expire等。在ehcache中，默认没有这两类事件的监听器，需要自主实现监听器以及监听器工厂类，然后配置到ehcache.xml中，方可生效。  
上述ehcache.xml配置中，给自定义cache都配置了cacheEventListenerFactory，用于监听缓存事件。同时也配置了cacheManager Factory实现类。具体实现代码如下  

CacheManagerEventListener简单实现  

	package com.yangyi.base.ehcache;

	import net.sf.ehcache.CacheException;
	import net.sf.ehcache.CacheManager;
	import net.sf.ehcache.Status;
	import net.sf.ehcache.event.CacheManagerEventListener;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;

	/**
	 * ehcache customer cacheManagerEventListener
	 * Created by yangjinfeng on 2017/1/5.
	 */
	public class CustomerCacheManagerEventListener implements CacheManagerEventListener {

	    private Logger logger = LoggerFactory.getLogger(getClass());

	    private final CacheManager cacheManager;

	    public CustomerCacheManagerEventListener(CacheManager cacheManager) {
	        this.cacheManager = cacheManager;
	    }

	    @Override
	    public void init() throws CacheException {
	        logger.info("init ehcache...");
	    }

	    @Override
	    public Status getStatus() {
	        return null;
	    }

	    @Override
	    public void dispose() throws CacheException {
	        logger.info("ehcache dispose...");
	    }

	    @Override
	    public void notifyCacheAdded(String s) {
	        logger.info("cacheAdded... {}", s);
	        logger.info(cacheManager.getCache(s).toString());
	    }

	    @Override
	    public void notifyCacheRemoved(String s) {

	    }
	}

CacheManagerEventListenerFactory的简单实现  

	package com.yangyi.base.ehcache;

	import net.sf.ehcache.CacheManager;
	import net.sf.ehcache.event.CacheManagerEventListener;
	import net.sf.ehcache.event.CacheManagerEventListenerFactory;

	import java.util.Properties;

	/**
	 * Created by yangjinfeng on 2017/1/5.
	 */
	public class CustomerCacheManagerEventListenerFactory extends CacheManagerEventListenerFactory {
	    @Override
	    public CacheManagerEventListener createCacheManagerEventListener(CacheManager cacheManager, Properties properties) {
	        return new CustomerCacheManagerEventListener(cacheManager);
	    }
	}


CacheEventListener的简单实现  

	package com.yangyi.base.ehcache;

	import net.sf.ehcache.CacheException;
	import net.sf.ehcache.Ehcache;
	import net.sf.ehcache.Element;
	import net.sf.ehcache.event.CacheEventListener;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;

	/**
	 * Created by yangjinfeng on 2017/1/5.
	 */
	public class CustomerCacheEventListener implements CacheEventListener {

	    private Logger logger = LoggerFactory.getLogger(getClass());

	    @Override
	    public void notifyElementRemoved(Ehcache ehcache, Element element) throws CacheException {
	        logger.info("cache removed. key = {}, value = {}", element.getObjectKey(), element.getObjectValue());
	    }

	    @Override
	    public void notifyElementPut(Ehcache ehcache, Element element) throws CacheException {
	        logger.info("cache put. key = {}, value = {}", element.getObjectKey(), element.getObjectValue());
	    }

	    @Override
	    public void notifyElementUpdated(Ehcache ehcache, Element element) throws CacheException {
	        logger.info("cache updated. key = {}, value = {}", element.getObjectKey(), element.getObjectValue());
	    }

	    @Override
	    public void notifyElementExpired(Ehcache ehcache, Element element) {
	        logger.info("cache expired. key = {}, value = {}", element.getObjectKey(), element.getObjectValue());
	    }

	    @Override
	    public void notifyElementEvicted(Ehcache ehcache, Element element) {
	        logger.info("cache evicted. key = {}, value = {}", element.getObjectKey(), element.getObjectValue());
	    }

	    @Override
	    public void notifyRemoveAll(Ehcache ehcache) {
	        logger.info("all elements removed. cache name = {}", ehcache.getName());
	    }

	    @Override
	    public Object clone() throws CloneNotSupportedException {
	        throw new CloneNotSupportedException();
	    }

	    @Override
	    public void dispose() {
	        logger.info("cache dispose.");
	    }
	}

CacheEventListenerFactory的简单实现  

	package com.yangyi.base.ehcache;

	import net.sf.ehcache.event.CacheEventListener;
	import net.sf.ehcache.event.CacheEventListenerFactory;

	import java.util.Properties;

	/**
	 * Created by yangjinfeng on 2017/1/5.
	 */
	public class CustomerCacheEventListenerFactory extends CacheEventListenerFactory {
	    @Override
	    public CacheEventListener createCacheEventListener(Properties properties) {
	        return new CustomerCacheEventListener();
	    }
	}

完成上述事件监听器的实现和配置后，我们可以启动应用，查看下相应事件监听器中输出的log，以验证是否生效  

	2017-01-07 10:27:07.810  INFO 4264 --- [           main] com.yangyi.Application                   : Starting Application on yangjinfeng-pc with PID 4264 (E:\JavaWorkSpace\spring-boot-ehcache3-hibernate5-jcache\target\classes started by yangjinfeng in E:\JavaWorkSpace\spring-boot-ehcache3-hibernate5-jcache)
	2017-01-07 10:27:07.810  INFO 4264 --- [           main] com.yangyi.Application                   : No active profile set, falling back to default profiles: default
	2017-01-07 10:27:07.865  INFO 4264 --- [           main] ationConfigEmbeddedWebApplicationContext : Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@2ef3eef9: startup date [Sat Jan 07 10:27:07 CST 2017]; root of context hierarchy
	2017-01-07 10:27:09.155  INFO 4264 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.transaction.annotation.ProxyTransactionManagementConfiguration' of type [class org.springframework.transaction.annotation.ProxyTransactionManagementConfiguration$$EnhancerBySpringCGLIB$$6eb4eae9] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
	2017-01-07 10:27:09.191  INFO 4264 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.cache.annotation.ProxyCachingConfiguration' of type [class org.springframework.cache.annotation.ProxyCachingConfiguration$$EnhancerBySpringCGLIB$$b7c72107] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
	2017-01-07 10:27:09.206  INFO 4264 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration' of type [class org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration$$EnhancerBySpringCGLIB$$ac3ae5ab] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
	2017-01-07 10:27:09.285  INFO 4264 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'spring.cache-org.springframework.boot.autoconfigure.cache.CacheProperties' of type [class org.springframework.boot.autoconfigure.cache.CacheProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
	2017-01-07 10:27:09.291  INFO 4264 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.cache.CacheManagerCustomizers' of type [class org.springframework.boot.autoconfigure.cache.CacheManagerCustomizers] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
	2017-01-07 10:27:09.293  INFO 4264 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.boot.autoconfigure.cache.EhCacheCacheConfiguration' of type [class org.springframework.boot.autoconfigure.cache.EhCacheCacheConfiguration$$EnhancerBySpringCGLIB$$46d9b2a9] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
	2017-01-07 10:27:09.418  INFO 4264 --- [           main] .y.b.e.CustomerCacheManagerEventListener : init ehcache...
	2017-01-07 10:27:09.487  INFO 4264 --- [           main] .y.b.e.CustomerCacheManagerEventListener : cacheAdded... org.hibernate.cache.spi.UpdateTimestampsCache
	2017-01-07 10:27:09.487  INFO 4264 --- [           main] .y.b.e.CustomerCacheManagerEventListener : [ name = org.hibernate.cache.spi.UpdateTimestampsCache status = STATUS_ALIVE eternal = true overflowToDisk = true maxEntriesLocalHeap = 5000 maxEntriesLocalDisk = 0 memoryStoreEvictionPolicy = LRU timeToLiveSeconds = 0 timeToIdleSeconds = 0 persistence = LOCALTEMPSWAP diskExpiryThreadIntervalSeconds = 120 cacheEventListeners: com.yangyi.base.ehcache.CustomerCacheEventListener ; orderedCacheEventListeners:  maxBytesLocalHeap = 0 overflowToOffHeap = false maxBytesLocalOffHeap = 0 maxBytesLocalDisk = 0 pinned = false ]
	2017-01-07 10:27:09.487  INFO 4264 --- [           main] .y.b.e.CustomerCacheManagerEventListener : cacheAdded... org.hibernate.cache.internal.StandardQueryCache
	2017-01-07 10:27:09.487  INFO 4264 --- [           main] .y.b.e.CustomerCacheManagerEventListener : [ name = org.hibernate.cache.internal.StandardQueryCache status = STATUS_ALIVE eternal = false overflowToDisk = true maxEntriesLocalHeap = 5 maxEntriesLocalDisk = 0 memoryStoreEvictionPolicy = LRU timeToLiveSeconds = 120 timeToIdleSeconds = 0 persistence = LOCALTEMPSWAP diskExpiryThreadIntervalSeconds = 120 cacheEventListeners: com.yangyi.base.ehcache.CustomerCacheEventListener ; orderedCacheEventListeners:  maxBytesLocalHeap = 0 overflowToOffHeap = false maxBytesLocalOffHeap = 0 maxBytesLocalDisk = 0 pinned = false ]
	2017-01-07 10:27:09.503  INFO 4264 --- [           main] .y.b.e.CustomerCacheManagerEventListener : cacheAdded... entityCache
	2017-01-07 10:27:09.503  INFO 4264 --- [           main] .y.b.e.CustomerCacheManagerEventListener : [ name = entityCache status = STATUS_ALIVE eternal = false overflowToDisk = true maxEntriesLocalHeap = 1000 maxEntriesLocalDisk = 10000 memoryStoreEvictionPolicy = LFU timeToLiveSeconds = 20 timeToIdleSeconds = 10 persistence = LOCALTEMPSWAP diskExpiryThreadIntervalSeconds = 120 cacheEventListeners: com.yangyi.base.ehcache.CustomerCacheEventListener ; orderedCacheEventListeners:  maxBytesLocalHeap = 0 overflowToOffHeap = false maxBytesLocalOffHeap = 0 maxBytesLocalDisk = 0 pinned = false ]
	2017-01-07 10:27:09.503  INFO 4264 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'ehCacheCacheManager' of type [class net.sf.ehcache.CacheManager] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
	2017-01-07 10:27:09.503  INFO 4264 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'cacheManager' of type [class org.springframework.cache.ehcache.EhCacheCacheManager] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
	2017-01-07 10:27:09.503  INFO 4264 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'cacheAutoConfigurationValidator' of type [class org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration$CacheManagerValidator] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
	2017-01-07 10:27:09.829  INFO 4264 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat initialized with port(s): 8080 (http)
	2017-01-07 10:27:09.839  INFO 4264 --- [           main] o.apache.catalina.core.StandardService   : Starting service Tomcat
	2017-01-07 10:27:09.839  INFO 4264 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/8.5.6
	2017-01-07 10:27:09.924  INFO 4264 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
	2017-01-07 10:27:09.924  INFO 4264 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 2061 ms

	2017-01-07 10:32:41.876  INFO 4264 --- [nio-8080-exec-1] c.y.b.e.CustomerCacheEventListener       : cache put. key = com.yangyi.entity.User#1, value = org.hibernate.cache.ehcache.internal.strategy.AbstractReadWriteEhcacheAccessStrategy$Item@1c13194d
	2017-01-07 10:32:41.877  INFO 4264 --- [nio-8080-exec-1] c.y.b.e.CustomerCacheEventListener       : cache put. key = com.yangyi.entity.Authority#1, value = org.hibernate.cache.ehcache.internal.strategy.AbstractReadWriteEhcacheAccessStrategy$Item@2accc177
	2017-01-07 10:32:41.878  INFO 4264 --- [nio-8080-exec-1] c.y.b.e.CustomerCacheEventListener       : cache put. key = com.yangyi.entity.Authority#2, value = org.hibernate.cache.ehcache.internal.strategy.AbstractReadWriteEhcacheAccessStrategy$Item@2b3b9c7e
	2017-01-07 10:32:41.879  INFO 4264 --- [nio-8080-exec-1] c.y.b.e.CustomerCacheEventListener       : cache put. key = com.yangyi.entity.Authority#3, value = org.hibernate.cache.ehcache.internal.strategy.AbstractReadWriteEhcacheAccessStrategy$Item@4c31e58c

## 注解方式使用二级缓存
要使用entity cache，需要在entity上配上相应的注解方可生效。javax.persistence.Cacheable注解标记该entity使用二级缓存，org.hibernate.annotations.Cache注解指定缓存策略，以及存放到哪个缓存区域。  
有关缓存策略详细信息可参考[hibernate5.0官方文档#缓存](http://docs.jboss.org/hibernate/orm/5.0/userguide/html_single/Hibernate_User_Guide.html#caching)

	package com.yangyi.entity;

	import org.hibernate.annotations.Cache;
	import org.hibernate.annotations.CacheConcurrencyStrategy;
	import javax.persistence.Cacheable;
	import javax.persistence.Entity;
	import javax.persistence.JoinTable;

	@Entity
	@Table(name = "users")
	@Cacheable
	@Cache(usage = CacheConcurrencyStrategy.READ_WRITE, region = "entityCache")
	public class User implements Serializable {

	}

最后，我们需要在spring boot应用层面打开cache功能，使用org.springframework.cache.annotation.EnableCaching注解  

	package com.yangyi;

	import org.springframework.boot.SpringApplication;
	import org.springframework.boot.autoconfigure.SpringBootApplication;
	import org.springframework.cache.annotation.EnableCaching;

	@SpringBootApplication
	@EnableCaching
	public class Application {

		public static void main(String[] args) {
			SpringApplication.run(Application.class, args);
		}
	}

## 完整代码
完整代码示例见github [spring-boot-ehcache2.x-hibernate-second-level-cache](https://github.com/yiduyangyi/cache/tree/master/spring-boot-ehcache2.x-hibernate-second-level-cache)
