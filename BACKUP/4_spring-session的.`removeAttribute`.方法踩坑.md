# [spring-session的 `removeAttribute` 方法踩坑](https://github.com/ParadiseWitch/gitblog/issues/4)

# 问题场景还原

**背景**：现在有一个服务使用了spring-session。服务有一个需求，需要在一个业务中将一个数据保存在用户session中的一个属性上，然后再在另一个业务将数据从redis中取出。类似以下模式：

1. 当用户操作第一次时，attr1作为key，数据1作为value会被添加到session中
2. 当用户操作第一次时，attr2作为key，数据2作为value会被添加到session中

……以此类推。以上是正常需求。

但是现在出现了一个问题：短时间内有一个用户操作次数过多，导致这个数据不断地在sessionMap中添加键值对，导致redis中这个人的对应的sessionMap越来越大，最后获取这个人的session时导致请求redis超时。

# 解决方案

这个问题其实算是设计问题，只要将原来将存在redis中的数据形式改为如下形式就可以：
```
key：sessionId+attrName1
value: 数据1
```

这样的话一条redis的数据也不会像sessionMap那样无限增加，造成请求超时。  

# 尝试基于原有设计进行修改

虽然上面的解决方案可以直接解决根本问题，但是我还是想基于原来的设计修改一下。

由于这个数据存过后只会使用一次，那么我在使用完成后在sessionMap中删除这个属性就好了

伪代码如下：

```java
// 唯一一次的获取session中这个属性的值
request.getSession().removeAttribute()
```

想法是好的，但是事与愿违。经过测试后发现，好家伙，`removeAttribute`方法确实把内存中的session中的属性删除了，但是更新到redis后，发现只是在sessionMap只把对应属性的value设置成了null：

![image](https://user-images.githubusercontent.com/37146904/231990280-c811d874-2904-4139-a6ea-68c8f8ec4454.png)

我对这样的行为感到奇怪，于是找到了一个issue：  https://github.com/spring-projects/spring-session/issues/1331

没办法，我尝试在获取一次session中这个属性的值之后就删除这个属性，代码like下面这样：

```java
// 唯一一次的获取session中这个属性的值
request.getSession().removeAttribute()
redisCacheHashService.sessionHmDelete(request.getSession().getId(), attrName)
```

`sessionHmDelete`是一个封装了一个使用`redisTemplate`直接删除redis中sessionMap中key的函数，代码类似下面这样：

```java
public void sessionHmDelete(string sessionId,String attrName){
	String redisKey = "spring:session:sessions:"+ sessionId;
	String mapKey = "sessionAttr:" + attrName;
	// get redisTemplate 
	RedisTemplate redisTemplate = redisTemplateService.getRedisTemplate(0);
	redisTemplate.setHashKeySerializer(new StringRedisSerializer());
	redisTemplate.sethashValueserializer(new JdkSerializationRedisSerializer());
	redisTemplate.afterPropertiesSet();
	redisTemplate.opsForHash().delete( redisKey, mapKey);
}
```

经过测试后发现通过这段代码后，redis依旧没有删除key，只是把value置为了null。

很奇怪，继续查看源码。

1. `removeAttribute`方法源码，可以看到这里`putAndFlush`第二个参数传入的为 null

    ![image](https://user-images.githubusercontent.com/37146904/231990280-c811d874-2904-4139-a6ea-68c8f8ec4454.png)

2. `putAndFlush`方法源码，其中`delta`是一个 map，临时记录改变的数据，方便等到更新时将所有变动更新到 redis 中。调用`removeAttribute`后，这个 map 就保存了一个键值对（key，null）

    ![image](https://user-images.githubusercontent.com/37146904/231990892-1a405e40-6392-4b84-8de0-03ea27c8a89b.png)

3. `flushImmediateIfNecessary`方法源码，更新时并不是马上一调用`removeAttribute`就会马上更新，默认时`RedisFlushMode`配置为`ON_SAVE`。
    
    ![image](https://user-images.githubusercontent.com/37146904/231991060-79d9dd9f-8a7a-4aa2-a2d8-8c03abbd44d0.png)

    
    
4. 由于不是立马更新到 redis，这个`saveDelta`方法会在我们手动使用 redisTemplate 删除 redis 中 session 中的属性后执行，于是此时又会将 delta 变动信息 put 到 redis 中，最终会将空值 put 到这个属性上。

    修改配置方法：
    ```xml
    <bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration">
      <property name="redisFlushMode" value="IMMEDIATE"/>
    </bean>
    ```

但是spring-session的配置不是说改就能改的，需要权衡可能对现有业务的影响。所以改配置也不是明智之举。

那么至于`RedisFlushMode`为`ON_SAVE`的情况下，什么时候才会调用`saveDelta`方法呢？这个问题我就没有深入研究下去了，暂时没有测试出来，那天有时间了再研究吧。


