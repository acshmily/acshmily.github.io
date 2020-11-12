---
title: Redis集群踩坑笔记
date: 2020-11-12 08:38:39
tags:
	-	Redis
	-	Springboot
---

#Redis集群踩坑笔记

## 背景

​		由于安全小组对于内网环境要求升级，对于原先的Redis集群需要进行IP过滤以及密码设置。由于之前开发同事部署集群的时候并未设置密码，选择了一个良辰吉日进行了密码设置然后重启了Redis服务，后面有意思的事情就来了T.T

## Spring Boot多集群配置

​		由于我们将缓存服务抽象成了一个服务层，这个服务层对应用服务器暴露标准接口进行数据存储调用，在Spring Boot2.3.3及以下版本由于RedisTemplate自动配置其实是存在缺陷的。

缺陷一：需要手动开启刷新拓扑配置

缺陷二：需要手动开启自动自适应

由于之前集群都并未设置密码，所以原先的代码片段为

```groovy diff
protected RedisConnectionFactory buildRedisConnectionFactory(RedisProperties redisProperty) {
        if (redisProperty == null) {
            //创建默认的redis连接工厂
            LettuceConnectionFactory defaultConnectionFactory = createDefaultConnectionFactory()
            defaultConnectionFactory.afterPropertiesSet()
            return defaultConnectionFactory
        }
        LettuceClientConfiguration clientConfig
        //创建lettuce连接工厂
        LettuceConnectionFactory redisConnectionFactory
        if (getRedisMode(redisProperty) == RedisModeEnum.CLUSTER) {
            //支持自适应集群拓扑刷新和静态刷新源

            ClusterTopologyRefreshOptions clusterTopologyRefreshOptions = ClusterTopologyRefreshOptions.builder()
                    .enablePeriodicRefresh()
                    .enableAdaptiveRefreshTrigger(ClusterTopologyRefreshOptions.RefreshTrigger.PERSISTENT_RECONNECTS, ClusterTopologyRefreshOptions.RefreshTrigger.MOVED_REDIRECT)
                    .build()
            ClusterClientOptions clusterClientOptions = ClusterClientOptions.builder().topologyRefreshOptions(clusterTopologyRefreshOptions).build()

            clientConfig = buildLettuceClientConfiguration(DefaultClientResources.create(),
                    redisProperty.getLettuce().getPool(), redisProperty, clusterClientOptions)
            redisConnectionFactory = new LettuceConnectionFactory(new RedisClusterConfiguration(redisProperty.getCluster().getNodes()), clientConfig)
        } else if (getRedisMode(redisProperty) == RedisModeEnum.SENTINEL) {
            clientConfig = buildLettuceClientConfiguration(DefaultClientResources.create(),
                    redisProperty.getLettuce().getPool(), redisProperty)
            redisConnectionFactory = new LettuceConnectionFactory(new RedisSentinelConfiguration
                    (redisProperty.getSentinel().getMaster(), new HashSet<>(redisProperty.getSentinel().getNodes())), clientConfig)
        } else {
            clientConfig = buildLettuceClientConfiguration(DefaultClientResources.create(),
                    redisProperty.getLettuce().getPool(), redisProperty)
            redisConnectionFactory = new LettuceConnectionFactory(new RedisStandaloneConfiguration
                    (redisProperty.getHost(), redisProperty.getPort()), clientConfig)
        }

        redisConnectionFactory.afterPropertiesSet()
        return redisConnectionFactory
    }
```

当Redis设置密码以后刷新Redis会一直提示

```verilog
NOAUTH Authentication required
```

后发现存在配置文件内设置的password并没有自动注入设置进去，需要手动设置，在Spring Boot 2.3.5后该问题依然未解决，但已经自动处理了自动刷新拓扑和自适应

```diff
protected RedisConnectionFactory buildRedisConnectionFactory(RedisProperties redisProperty) {
        if (redisProperty == null) {
            //创建默认的redis连接工厂
            LettuceConnectionFactory defaultConnectionFactory = createDefaultConnectionFactory()
            defaultConnectionFactory.afterPropertiesSet()
            return defaultConnectionFactory
        }
        LettuceClientConfiguration clientConfig
        //创建lettuce连接工厂
        LettuceConnectionFactory redisConnectionFactory
        if (getRedisMode(redisProperty) == RedisModeEnum.CLUSTER) {
            //支持自适应集群拓扑刷新和静态刷新源

            ClusterTopologyRefreshOptions clusterTopologyRefreshOptions = ClusterTopologyRefreshOptions.builder()
                    .enablePeriodicRefresh()
                    .enableAdaptiveRefreshTrigger(ClusterTopologyRefreshOptions.RefreshTrigger.PERSISTENT_RECONNECTS, ClusterTopologyRefreshOptions.RefreshTrigger.MOVED_REDIRECT)
                    .build()
            ClusterClientOptions clusterClientOptions = ClusterClientOptions.builder().topologyRefreshOptions(clusterTopologyRefreshOptions).build()

            clientConfig = buildLettuceClientConfiguration(DefaultClientResources.create(),
                    redisProperty.getLettuce().getPool(), redisProperty, clusterClientOptions)
+            RedisClusterConfiguration config
+            if(StringUtils.isNotBlank(redisProperty.password)){
+                config = new RedisClusterConfiguration(redisProperty.getCluster().getNodes())
+                log.info('设置Redis密码:{}',redisProperty.password)
+                config.setPassword(redisProperty.password)
+           }else{
+                config = new RedisClusterConfiguration(redisProperty.getCluster().getNodes())
+            }
-            redisConnectionFactory = new LettuceConnectionFactory(new RedisClusterConfiguration(redisProperty.getCluster().getNodes()), clientConfig)
+						 redisConnectionFactory = new LettuceConnectionFactory( config , clientConfig)
        } else if (getRedisMode(redisProperty) == RedisModeEnum.SENTINEL) {
            clientConfig = buildLettuceClientConfiguration(DefaultClientResources.create(),
                    redisProperty.getLettuce().getPool(), redisProperty)
            redisConnectionFactory = new LettuceConnectionFactory(new RedisSentinelConfiguration
                    (redisProperty.getSentinel().getMaster(), new HashSet<>(redisProperty.getSentinel().getNodes())), clientConfig)
        } else {
            clientConfig = buildLettuceClientConfiguration(DefaultClientResources.create(),
                    redisProperty.getLettuce().getPool(), redisProperty)
            redisConnectionFactory = new LettuceConnectionFactory(new RedisStandaloneConfiguration
                    (redisProperty.getHost(), redisProperty.getPort()), clientConfig)
        }

        redisConnectionFactory.afterPropertiesSet()
        return redisConnectionFactory
    }
```

## 是不是忘记了设置Redis集群的密码？

​		好了，Redis集群在代码侧配置可以正常启动并且查询数据了，但是后面遇到个诡异的现象。可以查数据，但是无法更新数据！Why？当时的定位思路：

- 是不是Redis连接工厂配置有问题？
- 查看RedisTemplate 的set操作有没有异常
- 检查集群是否正常

结论：

- 我进行了本地debug跟踪和复现，发现无法复现该问题，并且如果密码设置有问题的话应该也无法查出数据，排除该项
- 由于我们的服务对RedisTemplate进行了封装，基本都有try...catch，日志上并未体现异常捕获，故对于RedisTemplate来说操作都是正常的
- 那么，问题就应该出现在集群上了，经过与Redis负责部署维护的同事确认，他之前只设置了每个节点的Redis密码，并未设置**集群密码** ，后确认就是该问题导致出现了这么诡异的现象

## 结论

1. 对于RedisTemplate的操作是否成功现在不能完全依赖有没有报错，对于忘记设置集群密码的集群，RedisTemplate操作set并不会报错，这样会让人造成困惑，无法判断到底是哪儿出了问题
2. 不能盲目信任Spring Boot的AutoConfigure方法，有可能会忘记帮你set了password（单集群除外）
3. RTFM、STFW也不好使

