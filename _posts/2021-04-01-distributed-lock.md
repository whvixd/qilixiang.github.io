---
layout:     post
title:      分布式锁
subtitle:   
date:       2021-04-01
author:     Static
header-img: 
catalog: true
tags:
    - Java
    
---

## 1. What？

分布式锁，在分布式模型下，去争夺一个资源，先抢占资源的可以执行相关的业务逻辑，执行后再释放锁资源，未抢到锁资源的退出。

> 本文中利用redis（也可用mysql、zk等）实现


## 2. How？

> 利用spring aop实现分布式锁注解 `@DLock`，利用spring el实现幂等键

**分布式锁使用示例：**

```java
@RestController
public class PingController {

    @GetMapping("/ping")
    // 添加分布式锁
    @DLock(idempotentKeys = {"#id", "#type"})
    public BaseResult ping(@DKeyAlias("id")String id,@DKeyAlias("type")String type) {
        return Result.ok("hello world!");
    }
}
```

**AOP逻辑：**

```java
@Aspect
@Component
@Slf4j
public class DLockAspect {

    @Autowired
    private DLockManager DLockManager;


    @Pointcut("@annotation(com.github.whvixd.panic.buying.model.annotation.DLock)")
    public void dLockAspect() {
    }

    @Around("dLockAspect()")
    public Object lockAround(ProceedingJoinPoint proceeding) {
        DLockModel dLockModel = this.getDLockModel(proceeding);
        // 获取分布式锁
        boolean locked = this.getDLock(dLockModel);
        if (locked) {
            try {
                return proceeding.proceed();
            } catch (Throwable throwable) {
                log.error("DLockAspect->lockAround error,message:{} ", throwable.getMessage(), throwable);
                throw new BusinessException(BusinessExceptionCode.SYSTEM_ERROR.getErrorMessage(), throwable);
            } finally {
                if (dLockModel.getExpireType() == ExpireType.METHOD) {
                    // 删除锁
                    this.delLock(dLockModel);
                }
            }
        } else {
            log.info("DLockAspect->lockAround don't get dLock,threadName:{},dLockModel:{}", Thread.currentThread().getName(), dLockModel);
            throw new BusinessException(BusinessExceptionCode.NOT_GET_LOCK.getErrorMessage(), dLockModel.getLockPrefix() + "->" + dLockModel.getLockKey());
        }
    }

    /**
     * 获取分布式锁
     */
    private boolean getDLock(DLockModel dLockModel) {
        if (dLockModel == null) {
            return false;
        }
        String lockPrefix = dLockModel.getLockPrefix();
        String lockKey = dLockModel.getLockKey();
        String idempotentId = dLockModel.getIdempotentId();
        int timeToLiveSeconds = dLockModel.getTimeToLiveSeconds();
        if (StringUtils.isEmpty(lockPrefix) || StringUtils.isEmpty(lockKey)) {
            log.warn("DLockAspect->getDLock check false,lockPrefix:{},lockKey:{}", lockPrefix, lockKey);
            return false;
        }
        String redisKey = getRedisKey(lockPrefix, lockKey, idempotentId);
        return DLockManager.setnx(redisKey, "1", timeToLiveSeconds);
    }

    /**
     * 删除锁
     */
    private void delLock(DLockModel dLockModel) {
        if (dLockModel == null) {
            return;
        }
        String lockPrefix = dLockModel.getLockPrefix();
        String lockKey = dLockModel.getLockKey();
        String idempotentId = dLockModel.getIdempotentId();
        DLockManager.del(getRedisKey(lockPrefix, lockKey, idempotentId));
    }


    /**
     * 获取锁参数
     */
    private DLockModel getDLockModel(ProceedingJoinPoint proceeding) {
        MethodSignature sign = (MethodSignature) proceeding.getSignature();
        Method method = sign.getMethod();
        DLock dLock = method.getAnnotation(DLock.class);
        String lockPrefix = dLock.lockPrefix();
        String lockKey = dLock.lockKey();

        return DLockModel.builder()
                .lockPrefix(StringUtils.isBlank(lockPrefix) ? method.getDeclaringClass().getSimpleName() : lockPrefix)
                .lockKey(StringUtils.isBlank(lockKey) ? method.getName() : lockKey)
                .timeToLiveSeconds(Long.valueOf(dLock.timeUnit().toSeconds(dLock.expireTime())).intValue())
                .idempotentId(getIdempotentId(dLock.idempotentKeys(), proceeding.getArgs(), method.getParameters()))
                .expireType(dLock.expireType())
                .build();
    }

    /**
     * 获取EvaluationContext，填充模板
     */
    private EvaluationContext getEvaluationContext(Object[] args, Parameter[] parameters) {
        EvaluationContext context = new StandardEvaluationContext();
        IntStream.range(0, parameters.length).forEach(i -> {
            Parameter parameter = parameters[i];
            context.setVariable(getVariableName(parameter), args[i] == null ? StringUtils.EMPTY : args[i]);
        });
        return context;
    }

    /**
     * 获取变量名称
     */
    private String getVariableName(Parameter parameter) {
        if (parameter == null) {
            return StringUtils.EMPTY;
        }
        DKeyAlias annotation = parameter.getAnnotation(DKeyAlias.class);
        return annotation == null ? parameter.getName() :
                StringUtils.defaultIfBlank(annotation.value(), parameter.getName());
    }

    /**
     * 拼接幂等id
     */
    private String getIdempotentId(String[] idempotentKeys, Object[] args, Parameter[] parameters) {
        if (!ObjectUtils.allNotNull(idempotentKeys, args, parameters)) {
            return StringUtils.EMPTY;
        }
        EvaluationContext evaluationContext = getEvaluationContext(args, parameters);
        StringBuilder builder = new StringBuilder();
        for (String idempotentKey : idempotentKeys) {
            String elValue = String.valueOf(getElValue(evaluationContext, idempotentKey));
            if (StringUtils.isNotBlank(elValue)) {
                builder.append(elValue).append(":");
            }
        }
        String result = builder.toString();
        return StringUtils.isNotBlank(result) ? SecureUtil.md5(result) : result;
    }

    /**
     * 解析EL模板值
     */
    private Object getElValue(EvaluationContext context, String idempotentKey) {
        if (StringUtils.isBlank(idempotentKey) || context == null) {
            return StringUtils.EMPTY;
        }
        ExpressionParser parser = new SpelExpressionParser();
        Expression expression = parser.parseExpression(idempotentKey);
        try {
            return expression.getValue(context);
        } catch (EvaluationException e) {
            return StringUtils.EMPTY;
        }
    }

    /**
     * 获取redis key
     */
    private String getRedisKey(String lockPrefix, String lockKey, String idempotentId) {
        return lockPrefix + "-" + lockKey + (StringUtils.isNotBlank(idempotentId) ? "-" + idempotentId : StringUtils.EMPTY);
    }

}

```

**分布式锁管理器：**

> 需实现者自行注入IoC容器中

```java
public interface DLockManager {

    boolean setnx(String redisKey, String value, int timeToLiveSeconds);

    void del(String redisKey);
}
```

**分布式锁管理器示例：**

```java
public class DLockClientManager implements DLockManager {
    // 可使用redis、zk、mysql自行实现，本文中暂用本地cache实现
    private Cache<String, Object> guavaCache = CacheBuilder.newBuilder().expireAfterWrite(5, TimeUnit.SECONDS).recordStats().initialCapacity(100).build();

    @Override
    public boolean setnx(String redisKey, String value, int timeToLiveSeconds) {
        Object object = guavaCache.getIfPresent(redisKey);
        if (object != null) {
            return false;
        }
        guavaCache.put(redisKey, value);
        log.info("RedisClientManager->setnx success,redisKey:{},value:{},timeToLiveSeconds:{}", redisKey, value, timeToLiveSeconds);
        return true;
    }

    @Override
    public void del(String redisKey) {
        guavaCache.invalidate(redisKey);
        log.info("RedisClientManager->del success,redisKey:{}", redisKey);
    }
}
```

**测试：**

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@Slf4j
public class DLockAspectTest {
    @Autowired
    private PingController pingController;

    @Test(expected = BusinessException.class)
    public void lockAroundTest() throws InterruptedException {
        // 占用锁资源时间默认为5s，使用者可配置化
        pingController.ping();
        log.info("-------");
        pingController.ping();
        Thread.sleep(10000L);
    }

}
```