---
title: Springboot多缓存配置处理
date: 2017-08-21 16:01:48
comments: true
tags:
- springboot
- spring
- springmvc
- 随笔
---

* 下面我们已Ehcache,Redis缓存配置为例讲解。

### 1. Ehcache缓存配置
```java
@Configuration
@EnableCaching
public class EhCacheConfig {

    /**
     * EhCache的配置
     */
	@Primary
    @Bean(name="ehcacheManager")
    public EhCacheCacheManager cacheManager(CacheManager cacheManager) {
        return new EhCacheCacheManager(cacheManager);
    }

    /**
     * EhCache的配置
     */
    @Bean
    public EhCacheManagerFactoryBean ehcache() {
        EhCacheManagerFactoryBean ehCacheManagerFactoryBean = new EhCacheManagerFactoryBean();
        ehCacheManagerFactoryBean.setConfigLocation(new ClassPathResource("ehcache.xml"));
        return ehCacheManagerFactoryBean;
    }
}
```
<!-- more -->

### 2. Redis缓存配置

```java
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {

    @Bean
    public KeyGenerator keyGenerator(){
        return (Object target, Method method, Object... params)->{
            StringBuilder sb = new StringBuilder();
            sb.append(target.getClass().getName());
            sb.append(method.getName());
            for (Object obj : params) {
                sb.append(obj.toString());
            }
            return sb.toString();
        };
    }

    @SuppressWarnings("rawtypes")
	@Bean(name="redisManager")
    public CacheManager cacheManager(RedisTemplate redisTemplate){
        RedisCacheManager rcm = new RedisCacheManager(redisTemplate);
        return rcm;
    }

    @Bean
    public RedisTemplate<String,String> redisTemplate(RedisConnectionFactory factory){
        StringRedisTemplate template = new StringRedisTemplate(factory);
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<Object>(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        template.setValueSerializer(jackson2JsonRedisSerializer);
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }
}
```
> `@Primary`注解代表默认情况下用Ehcache缓存数据信息,你也可以把它加在RedisManager配置上，代表默认已Redis缓存数据信息。

### 3.使用缓存存储相关数据
```java
@CacheConfig(cacheManager="ehcacheManager")
public interface IConstantFactory {

    /**
     * 通过角色ids获取角色名称
     */
    @Cacheable(value = Cache.CONSTANT, key = "'" + CacheKey.ROLES_NAME + "'+#roleIds")
    String getRoleName(String roleIds);

    /**
     * 通过角色id获取角色名称
     */
    @Cacheable(value = Cache.CONSTANT, key = "'" + CacheKey.SINGLE_ROLE_NAME + "'+#roleId")
    String getSingleRoleName(Integer roleId);

}
```
> `@CacheConfig`注解中cacheManager配置的是当前类使用的缓存构造器。
> `@Cacheable`注解配置当前查询方法以什么规则缓存数据。

* 本篇文章主要讲解下springboot多缓存应该怎么配置和使用，其他内容没有过多介绍，有兴趣同学可以自行搜索学习。这篇[Spring Boot中的缓存支持注解配置与EhCache使用](http://blog.didispace.com/springbootcache1/)可以参考查看。