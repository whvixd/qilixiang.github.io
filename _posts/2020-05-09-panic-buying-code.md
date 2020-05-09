---
layout:     post
title:      秒杀系统实践
subtitle:   panic-buying
date:       2020-05-09
author:     Static
header-img: img/bg/black.jpg
catalog: true
tags:
    - panic-buying
    
---

> 项目源码:[https://github.com/whvixd/panic-buying](https://github.com/whvixd/panic-buying)

## 1. 架构图

<html>
    <img src="/img/panic-buying/framework.jpg" width="600" height="600" /> 
</html>

> 只实现秒杀的后端部分

---

## 2. 技术点
1. 框架:**SpringBoot**
2. 代码管理:**git**
3. 项目结构管理:**maven**
4. 限流:guava的**RateLimiter**工具，@see 4.2
5. 缓存:**guavaCache**，@see 4.3
6. 消息队列:jdk自带的**LinkedBlockingQueue**，@see 4.4
7. 数据库:**h2**，@see 4.5
8. 压力测试:**jmeter** @see 4.6

---

## 3. 项目结构

<html>
    <img src="/img/panic-buying/catalog.jpg" width="400" height="400" /> 
</html>

---

## 4. 源码分析

#### 4.1 订单创建接口层
异步创建订单，添加限流控制和订单数量限制，具体实现@see 4.2 4.3

```java
@RestController
@RequestMapping("/sale/order")
public class SaleOrderController {

    @Autowired
    private SaleOrderService saleOrderService;

    @PostMapping("/create")
    // 自定义注解，限流，每秒创建10个令牌
    @RateLimit(permitsPerSecond = 10)
    public Result create(@RequestBody SaleOrderVO.Arg arg) {
        try {
        	// 异步创建订单，推送到队列中
            saleOrderService.asyncCreate(arg.getProductId());
            return Result.ok();
        } catch (Exception e) {
            return Result.fail(e.getMessage());
        }
    }
}
```

#### 4.2 限流实现

利用spring aop，在请求进入接口之前，拦截添加了**@RateLimit**注解的方法，校验请求速度

```java
@Slf4j
@Aspect
@Component
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class RateLimitAspect {

	// key:className_methodName,value:RateLimiter
    private Map<String, RateLimiter> rateLimiterMap = Maps.newConcurrentMap();

	// 动态代理，在请求进入接口之前，拦截添加 @RateLimit 注解的方法，校验请求速度
    @Before(value = "@annotation(com.github.whvixd.panic.buying.model.annotation.RateLimit)")
    public void limit(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        String className = joinPoint.getTarget().getClass().getName();
        String methodName = method.getName();
        String key = className + "_" + methodName;

        RateLimiter rateLimiter = rateLimiterMap.get(key);
        RateLimit rateLimit = method.getDeclaredAnnotation(RateLimit.class);
        double permitsPerSecond = rateLimit.permitsPerSecond();
        int permits = rateLimit.permits();
        if (rateLimiter == null) {
            rateLimiter = RateLimiter.create(permitsPerSecond);
            rateLimiterMap.put(key, rateLimiter);
        }
        log.info("rate limit method:{}", key);
        rateLimiter.acquire(permits);
    }
}
```
#### 4.3 订单数量限制
利用spring aop 对创建订单的方法进行控制，a. 在进入接口之前，将产品id保存到本地缓存中，并自增;b. 同一线程在创建订单接口结束后自减

> 自增、自减操作是线程安全的，添加了锁机制

```java
@Slf4j
@Aspect
@Component
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class SaleOrderInterfaceAspect {

    @Autowired
    private CacheManager cacheManager;

    @Before("execution(public * com.github.whvixd.panic.buying.controller.SaleOrderController.create(*))")
    public void lock(JoinPoint joinPoint) {
        Object[] args = joinPoint.getArgs();
        if (args[0] instanceof SaleOrderVO.Arg) {
            SaleOrderVO.Arg arg = (SaleOrderVO.Arg) args[0];
            Long productId = arg.getProductId();
            // 进入创建订单方法之前，根据产品id，自增
            int count = cacheManager.add(productId);
            log.info("add count:{}", count);
            // 若大于配置的数量，直接异常
            if (count < 0 || count > 100) {
                throw new RuntimeException();
            }
        }
    }

    @After("execution(public * com.github.whvixd.panic.buying.controller.SaleOrderController.create(*))")
    public void unLock(JoinPoint joinPoint) {
        Object[] args = joinPoint.getArgs();
        if (args[0] instanceof SaleOrderVO.Arg) {
            SaleOrderVO.Arg arg = (SaleOrderVO.Arg) args[0];
            Long productId = arg.getProductId();
            // 请求结束后，自减
            int count = cacheManager.subtract(productId);
            log.info("subtract count:{}", count);
        }
    }
}
```

#### 4.4 消息队列

- **提供者**

```java
@Component
public class SaleOrderProducer {
    @Autowired
    private BlockQueueManager blockQueueManager;

    private final Lock lock = Lock.instance;

    @Async
    public void send(Object message) {
        synchronized (lock) {
        	// 推送消息
            blockQueueManager.put(message);
            // 当有订单进入时，唤醒消费者线程消费订单
            lock.notifying();
        }
    }
}
```

- **消费者**

```java
@Component
@Slf4j
public class SaleOrderConsumer {
	// 锁
    private final Lock lock = Lock.instance;
    @Autowired
    private BlockQueueManager blockQueueManager;

    @Autowired
    private SaleOrderService saleOrderService;

    // 服务启动时，再启动一个线程作为消费者
    @PostConstruct
    public void start() {
        InvokeTask.newInstance(this::create).invokeTaskName("SaleOrderConsumerThread").start();
    }

    // 消费者监控队列是否有订单，若有订单，则创建订单，否则线程等待
    private void create() {
        synchronized (lock) {
            for (;;) {
                try {
                	// 队列为空，线程等待，唤起线程逻辑在提供者推送订单时
                    if (blockQueueManager.isEmpty()) {
                        lock.waiting();
                    }
                    Object o = blockQueueManager.pull();
                    Long productId;
                    if (o instanceof Long) {
                        productId = (Long) o;
                    } else {
                        log.warn("class not match,element class:{}", o.getClass().getName());
                        continue;
                    }

                    // 事务创建订单:1. 更新产品订单中已购的数量，再去校验是否超卖；2. 若没有超卖则创建订单
                    saleOrderService.create(productId);
                    log.info("consumer create success,productId:{}", productId);
                } catch (Exception e) {
                    log.warn("SaleOrderConsumer thread waiting from ", e);
                    try {
                        lock.waiting();
                    } catch (InterruptedException interruptedException) {
                        log.error("SaleOrderConsumer InterruptedException ", interruptedException);
                        throw new RuntimeException(e);
                    }
                }
            }
        }
    }
}
```

#### 4.5 h2

当服务启动时，自动创建产品和订单表，并在产品表中添加 "小米" 产品，数量为100

```sql
-- 新建表
-- 产品
DROP TABLE IF EXISTS PRODUCT;
CREATE TABLE PRODUCT (
  PRODUCT_ID      INT          NOT NULL PRIMARY KEY AUTO_INCREMENT,
  -- 产品名称
  PRODUCT_NAME    VARCHAR(100) NOT NULL,
  -- 总数 
  PRODUCT_TOTAL   INT          NOT NULL,
  -- 已购数
  PRODUCT_SOLD    INT          NOT NULL,
  -- 版本号
  PRODUCT_VERSION INT
);

-- 订单
DROP TABLE IF EXISTS SALE_ORDER;
CREATE TABLE SALE_ORDER (
  ORDER_ID    INT             NOT NULL PRIMARY KEY AUTO_INCREMENT,
  -- 产品
  PRODUCT_ID  INT             NOT NULL,
  -- 订单名称
  ORDER_NAME  VARCHAR(100)    NOT NULL,
  -- 创建时间
  CREATE_TIME LONGVARCHAR(20) NOT NULL
);

-- 插入 "小米"产品，总数100
INSERT INTO PRODUCT (PRODUCT_NAME,PRODUCT_TOTAL,PRODUCT_SOLD) VALUES('小米',100,0);
```

#### 4.6 测试

**单元测试**

```java
@Slf4j
@RunWith(SpringRunner.class)
@SpringBootTest
@WebAppConfiguration
public class SaleOrderControllerTest extends BaseTest {

    @Autowired
    private SaleOrderController saleOrderController;

    private MockMvc mockMvc;

    private ObjectMapper mapper = new ObjectMapper();

    @Rule
    public ContiPerfRule contiPerfRule = new ContiPerfRule();

    @Before
    public void setupMockMvc() throws Exception {
        mockMvc = MockMvcBuilders.standaloneSetup(saleOrderController).build();
    }

    @Test
    // 创建20个线程，总共调用创建订单接口200次
    @PerfTest(invocations = 200, threads = 20)
    public void restCreateTest() throws Exception {
        SaleOrderVO.Arg arg = new SaleOrderVO.Arg();
        arg.setProductId(1L);

        mockMvc.perform(MockMvcRequestBuilders.post("/sale/order/create")
                .contentType(MediaType.APPLICATION_JSON_UTF8)
                .content(mapper.writeValueAsString(arg)))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.content().contentType(MediaType.APPLICATION_JSON_UTF8))
                .andDo(MockMvcResultHandlers.print());
    }
}
```
- **jmeter 请求接口**
<html>
    <img src="/img/panic-buying/jmeter.jpg" width="600" height="600" /> 
</html>

- **后端日志**
<html>
    <img src="/img/panic-buying/error_log.jpg" width="600" height="400" /> 
</html>

> 当超过100时，跑出异常，提示超卖

- **数据库**
<html>
    <img src="/img/panic-buying/data.jpg" width="600" height="600" /> 
</html>

**项目源码**:[https://github.com/whvixd/panic-buying](https://github.com/whvixd/panic-buying)