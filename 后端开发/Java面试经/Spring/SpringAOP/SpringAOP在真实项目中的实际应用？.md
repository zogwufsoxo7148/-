今天，Java已经卷到“屎上雕花”的程度，八股文的准备如果仅仅靠背诵，很容易陷入“背了忘，忘了背”的死循环中。

所以，我们必须：结合具体的代码demo，尝试系统地掌握，才能更好的卷出一条活路。

#### 1. **日志记录**

记录方法的执行时间、参数和返回值。

```Java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object proceed = joinPoint.proceed();
        long executionTime = System.currentTimeMillis() - start;

        System.out.println(joinPoint.getSignature() + " executed in " + executionTime + "ms");
        return proceed;
    }
}

@Service
public class MyService {
    public void perform() {
        System.out.println("Performing service...");
    }
}
```

#### 2. **权限检查**

在方法执行前进行权限验证，防止未经授权的访问。

```Java
@Aspect
@Component
public class SecurityAspect {

    @Before("execution(* com.example.service.*.*(..)) && @annotation(com.example.security.Secured)")
    public void checkSecurity(JoinPoint joinPoint) {
        // 这里进行权限检查逻辑
        System.out.println("Checking security for " + joinPoint.getSignature().getName());
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Secured {
    String role() default "USER";
}

@Service
public class MyService {

    @Secured(role = "ADMIN")
    public void performAdminTask() {
        System.out.println("Performing admin task...");
    }

    public void perform() {
        System.out.println("Performing service...");
    }
}
```

#### 3. **事务管理**

自动管理事务的开启、提交和回滚。

```Java
@Aspect
@Component
public class TransactionAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object manageTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("Transaction start");
        try {
            Object result = joinPoint.proceed();
            System.out.println("Transaction commit");
            return result;
        } catch (Exception ex) {
            System.out.println("Transaction rollback due to: " + ex.getMessage());
            throw ex;
        }
    }
}

@Service
public class MyService {

    public void performTransaction() {
        System.out.println("Performing transactional service...");
        // 模拟异常
        if (true) throw new RuntimeException("Transaction failed");
    }
}
```

#### 4. **异常处理**

统一处理方法执行时的异常。

```Java
@Aspect
@Component
public class ExceptionHandlingAspect {

    @AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "ex")
    public void handleException(JoinPoint joinPoint, Exception ex) {
        System.out.println("Exception in " + joinPoint.getSignature().getName() + " with message: " + ex.getMessage());
        // 这里可以记录日志、通知管理员等
    }
}

@Service
public class MyService {

    public void perform() {
        System.out.println("Performing service...");
        throw new RuntimeException("Unexpected error");
    }
}
```

#### 5. **缓存管理**

在方法执行前检查缓存，在执行后更新缓存。

```Java
@Aspect
@Component
public class CachingAspect {

    private Map<String, Object> cache = new HashMap<>();

    @Around("execution(* com.example.service.*.*(..)) && @annotation(com.example.cache.Cacheable)")
    public Object cacheResult(ProceedingJoinPoint joinPoint) throws Throwable {
        String key = joinPoint.getSignature().toString();
        if (cache.containsKey(key)) {
            System.out.println("Returning cached result for " + key);
            return cache.get(key);
        }

        Object result = joinPoint.proceed();
        cache.put(key, result);
        System.out.println("Caching result for " + key);
        return result;
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Cacheable {
}

@Service
public class MyService {

    @Cacheable
    public String getData() {
        System.out.println("Fetching data from database...");
        return "Data from DB";
    }
}
```

### 总结：

Spring AOP 在实际项目中广泛用于日志记录、权限检查、事务管理、异常处理和缓存管理等场景。这些示例展示了如何通过切面简化和增强应用程序的业务逻辑。

Coding不易，棒棒，加油！