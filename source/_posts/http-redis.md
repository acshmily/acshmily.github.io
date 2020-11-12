---
title: http-redis
date: 2020-05-08 08:45:36
tags:
	- redis
categories: 工作笔记

---



# 内存持久化方案

## 前提

​		方案底层不限于Redis、FastCache、Memcache。应剥离及封装底层；对外暴露为HTTP接口或Socket接口进行能力封装；

​		键统一采用"prefix_主叫号码"风格进行约束

## 方案

###方案一 JSON序列化

优点：序列化后在终端内查看直观

缺点：全部业务系统统一采用同一个JSON对象进行序列化及反序列化，拓展性不好

###方案二 采用特定约束字符串分隔

优点：无限拓展（只要定义好顺序）

缺点：终端内不直观

## 持久化测试对比

将同等数量级的数据进行以上两组的序列化及反序列化调用测试QPS时间

约定JSON对象

```groovy
class beans{
  String type // 开关
  String prd1 // 产品功能1
  String prd2 // 产品功能2
  String prd3 // 产品功能3
  String prd4 // 产品功能4
  String prd5 // 产品功能5
  String prd6 // 产品功能6
}
```

约定特定分隔符

```
type|prd1|prd2|prd3|prd4|prd5|prd6
```

测试代码

```groovy
/**
 @Author: Huanghz
 @Description: TODO
 @Date:Created in 10:03 上午 2020/5/7
 @ModifyBy:
  * */
@ToString
class Beans implements RedisSerializerType {


    String type
    boolean prd1
    String prd2
    boolean prd3
    boolean prd4
    boolean prd5
    boolean prd6

    Beans(String type, boolean prd1, String prd2, boolean prd3, boolean prd4, boolean prd5, boolean prd6) {
        this.type = type
        this.prd1 = prd1
        this.prd2 = prd2
        this.prd3 = prd3
        this.prd4 = prd4
        this.prd5 = prd5
        this.prd6 = prd6
    }

    Beans() {
    }

    String toRedisValue() {
        def sb = new StringBuffer()
        this.class.declaredFields.findAll { !it.synthetic && !Modifier.isStatic(it.modifiers) }.each {
            sb.append(this."$it.name").append('|')
        }
        sb.deleteCharAt(sb.lastIndexOf('|'))
        return sb.toString()
    }

    static Beans getInstance(String redisValue) {
        instance = new Beans()
        def point = 0
        def values = redisValue.split('\\|')
        instance.class.declaredFields.findAll { !it.synthetic && !Modifier.isStatic(it.modifiers) }.each {
            it.setAccessible(true)
            switch (it.type.name) {
                case 'boolean':
                    instance.setProperty(it.name,values[point].toUpperCase() == 'TRUE')
                    break
                default:
                    instance.setProperty(it.name,values[point])
            }
            point++
        }
        return instance
    }
    private static Beans instance
}

package cn._118114.hangup

import cn._118114.hangup.beans.RedisSerializerType
import groovy.transform.ToString
import groovy.util.logging.Slf4j;
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.data.redis.core.RedisTemplate
import org.springframework.data.redis.core.StringRedisTemplate
import org.junit.jupiter.api.Test

import javax.annotation.Resource
import java.util.concurrent.TimeUnit;

/**
 * @Author: Huanghz
 * @Description: Json序列化和反序列化测试
 * @Date:Created in 4:54 下午 2020/5/6
 * @ModifyBy:
 * */
@SpringBootTest
@Slf4j
class JsonTest {
    final static _PREFIX = 'json_'
    // 定义beans

    /**
     * 处理
     */
    @Test
    void flush() {
        def beans = new Beans()
        beans.type = 'test'
        beans.prd2 = 'prd2'
        log.info('开始循环插入Redis')
        for (int i = 0; i < 100000; i++) {
            redisTemplate.boundValueOps(_PREFIX + i).setIfAbsent(beans, 5, TimeUnit.MINUTES)
        }
        log.info('开始从Redis反序列化')
        def start = System.currentTimeMillis()
        for (int i = 0; i < 100000; i++) {
            beans = redisTemplate.boundValueOps(_PREFIX + i).get()
        }
        def end = System.currentTimeMillis()
        log.info('读取结束，消耗时间:{}', end - start)


    }

    @Test
    void flush1() {
        def beans = new Beans()
        beans.type = 'test'
        beans.prd2 = 'prd2'
        for (int i = 100000; i < 200000; i++) {
            redisTemplate.boundValueOps(_PREFIX + i).setIfAbsent(beans.toRedisValue(), 5, TimeUnit.MINUTES)
        }
        log.info('开始从Redis反序列化')
        def start = System.currentTimeMillis()
        for (int i = 100000; i < 200000; i++) {
            beans = Beans.getInstance(redisTemplate.boundValueOps(_PREFIX + i).get() as String)
        }
        def end = System.currentTimeMillis()
        log.info('读取结束，消耗时间:{}', end - start)
    }
    @Resource
    private RedisTemplate redisTemplate
}

```

### JSON序列化方案耗时

测试10W条序列化反序列化耗时日志

```verilog
2020-05-07 13:34:19.593  INFO 7458 --- [           main] cn._118114.hangup.JsonTest               : 开始循环插入Redis
2020-05-07 13:34:20.425  INFO 7458 --- [           main] io.lettuce.core.EpollProvider            : Starting without optional epoll library
2020-05-07 13:34:20.428  INFO 7458 --- [           main] io.lettuce.core.KqueueProvider           : Starting without optional kqueue library
2020-05-07 13:34:33.779  INFO 7458 --- [           main] cn._118114.hangup.JsonTest               : 开始从Redis反序列化
2020-05-07 13:34:43.346  INFO 7458 --- [           main] cn._118114.hangup.JsonTest               : 读取结束，消耗时间:9551




```

### 自定义约定序列化耗时

```verilog
2020-05-07 10:52:22.303  INFO 6554 --- [           main] io.lettuce.core.EpollProvider            : Starting without optional epoll library
2020-05-07 10:52:22.311  INFO 6554 --- [           main] io.lettuce.core.KqueueProvider           : Starting without optional kqueue library
2020-05-07 10:52:41.667  INFO 6554 --- [           main] cn._118114.hangup.JsonTest               : 开始从Redis反序列化
2020-05-07 10:52:56.940  INFO 6554 --- [           main] cn._118114.hangup.JsonTest               : 读取结束，消耗时间:15256


```

## 基于HTTP 封装内存数据库

GitHub测试项目地址：https://github.com/acshmily/http-redis

测试框架：

​	Spring Boot 2.2.6、Redis连接池 Luttuce 

配置文件：

```yaml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    database: 0
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0

```

压测结果：

{% asset_img jmeter.png jmeter压测结果 %}