---
title: Spring之InitializingBean接口
date: 2019-03-17 09:51:41
tags: Spring
categories: Spring
---




### 什么时候使用
有时候会遇到这样的问题：
在我们将一个Bean交给Spring管理的时候，有时候我们的Bean中有某个属性需要注入，但是又不能通过一般的方式注入，什么意思呢？举个栗子：首先我们有个Service,在该Service中有一个属性，但是该属性不支持Spring注入，只能通过Build或者new的方式创建（比如StringBuffer之类的），但是我们想在Spring配置Bean的时候一起将该属性注入进来，这时候该怎么办呢？这时候可以通过实现InitializingBean接口来解决！


### 源代码
```java
package org.springframework.beans.factory;

public abstract interface InitializingBean
{
  public abstract void afterPropertiesSet()
    throws Exception;
}

/* Location:           C:\Users\Administrator\.m2\repository\org\springframework\spring-beans\4.3.13.RELEASE\spring-beans-4.3.13.RELEASE.jar
 * Qualified Name:     org.springframework.beans.factory.InitializingBean
 * Java Class Version: 6 (50.0)
 * JD-Core Version:    0.7.0.1
 */
```
```java
/*     */ package org.springframework.cache.support;
/*     */ 
/*     */ import java.util.Collection;
/*     */ import java.util.Collections;
/*     */ import java.util.LinkedHashSet;
/*     */ import java.util.Set;
/*     */ import java.util.concurrent.ConcurrentHashMap;
/*     */ import java.util.concurrent.ConcurrentMap;
/*     */ import org.springframework.beans.factory.InitializingBean;
/*     */ import org.springframework.cache.Cache;
/*     */ import org.springframework.cache.CacheManager;

/
/*     */ public abstract class AbstractCacheManager
/*     */   implements CacheManager, InitializingBean
/*     */ {
/*  41 */   private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap(16);
/*     */   
/*  43 */   private volatile Set<String> cacheNames = Collections.emptySet();
/*     */   
/*     */ 
/*     */   public AbstractCacheManager() {}
/*     */   
/*     */   public void afterPropertiesSet()
/*     */   {
/*  50 */     initializeCaches();
/*     */   }

/*     */   public void initializeCaches()
/*     */   {
/*  61 */     Collection<? extends Cache> caches = loadCaches();
/*     */     
/*  63 */     synchronized (this.cacheMap) {
/*  64 */       this.cacheNames = Collections.emptySet();
/*  65 */       this.cacheMap.clear();
/*  66 */       Set<String> cacheNames = new LinkedHashSet(caches.size());
/*  67 */       for (Cache cache : caches) {
/*  68 */         String name = cache.getName();
/*  69 */         this.cacheMap.put(name, decorateCache(cache));
/*  70 */         cacheNames.add(name);
/*     */       }
/*  72 */       this.cacheNames = Collections.unmodifiableSet(cacheNames);
/*     */     }
/*     */   }

/*     */   protected abstract Collection<? extends Cache> loadCaches();
/*     */   
/*     */ 
/*     */ 
/*     */ 
/*     */ 
/*     */   public Cache getCache(String name)
/*     */   {
/*  88 */     Cache cache = (Cache)this.cacheMap.get(name);
/*  89 */     if (cache != null) {
/*  90 */       return cache;
/*     */     }
/*     */     
/*     */ 
/*  94 */     synchronized (this.cacheMap) {
/*  95 */       cache = (Cache)this.cacheMap.get(name);
/*  96 */       if (cache == null) {
/*  97 */         cache = getMissingCache(name);
/*  98 */         if (cache != null) {
/*  99 */           cache = decorateCache(cache);
/* 100 */           this.cacheMap.put(name, cache);
/* 101 */           updateCacheNames(name);
/*     */         }
/*     */       }
/* 104 */       return cache;
/*     */     }
/*     */   }
/*     */   
/*     */ 
/*     */   public Collection<String> getCacheNames()
/*     */   {
/* 111 */     return this.cacheNames;
/*     */   }
/*     */   

/*     */ 
/*     */   protected final Cache lookupCache(String name)
/*     */   {
/* 128 */     return (Cache)this.cacheMap.get(name);
/*     */   }
/*     */   
/*     */ 
/*     */ 
/*     */ 
/*     */ 
/*     */   @Deprecated
/*     */   protected final void addCache(Cache cache)
/*     */   {
/* 138 */     String name = cache.getName();
/* 139 */     synchronized (this.cacheMap) {
/* 140 */       if (this.cacheMap.put(name, decorateCache(cache)) == null) {
/* 141 */         updateCacheNames(name);
/*     */       }
/*     */     }
/*     */   }

/*     */   private void updateCacheNames(String name)
/*     */   {
/* 154 */     Set<String> cacheNames = new LinkedHashSet(this.cacheNames.size() + 1);
/* 155 */     cacheNames.addAll(this.cacheNames);
/* 156 */     cacheNames.add(name);
/* 157 */     this.cacheNames = Collections.unmodifiableSet(cacheNames);
/*     */   }
/*     */   

/*     */   protected Cache decorateCache(Cache cache)
/*     */   {
/* 170 */     return cache;
/*     */   }
/*     */   

/*     */   protected Cache getMissingCache(String name)
/*     */   {
/* 187 */     return null;
/*     */   }
/*     */ }

/* Location:           C:\Users\Administrator\.m2\repository\org\springframework\spring-context\4.3.13.RELEASE\spring-context-4.3.13.RELEASE.jar
 * Qualified Name:     org.springframework.cache.support.AbstractCacheManager
 * Java Class Version: 6 (50.0)
 * JD-Core Version:    0.7.0.1
 */
```
### 初始化bean的两种方式
1. 实现InitializingBean接口，实现afterPropertiesSet方法，或者在配置文件中同过init-method指定，两种方式可以同时使用

2. 实现InitializingBean接口是直接调用afterPropertiesSet方法，比通过反射调用init-method指定的方法效率相对来说要高点。但是init-method方式消除了对spring的依赖

3. 如果调用afterPropertiesSet方法时出错，则不调用init-method指定的方法。

### 参考
<https://blog.csdn.net/flqljh/article/details/49834541>
