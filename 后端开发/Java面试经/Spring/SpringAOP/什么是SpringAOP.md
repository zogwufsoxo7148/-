今天，Java已经卷到“屎上雕花”的程度，八股文的准备如果仅仅靠背诵，很容易陷入“背了忘，忘了背”的死循环中。

所以，我们必须：结合具体的代码demo，尝试系统地掌握，才能更好的卷出一条活路。

Spring AOP（Aspect-Oriented Programming，面向切面编程）是Spring框架中的一个重要模块，旨在**通过将**横切关注点**（**cross-cutting concerns**）从业务逻辑中分离出来，从而实现代码的模块化和重用。**AOP允许开发者定义“切面”（aspects），这些切面可以应用于多个目标对象（例如类或方法），以减少代码重复、提高可维护性，并增强代码的可读性。

### 核心概念

1. **切面（Aspect）**：
   1. **切面是一个模块化的关注点**，通常代表了横切关注点，如日志记录、事务管理、权限控制等。切面可以看作是横切多个对象的功能，它**将关注点与业务逻辑分离开来。**
2. **连接点（Join Point）**：
   1. 连接点是程序执行中的某个具体点，例如方法的调用、方法的执行、字段的访问等。**在Spring** **AOP****中，连接点通常指的是方法的执行点。**
3. **切入点（Pointcut）**：
   1. 切入点是定义在哪些连接点上应用切面的表达式。**通过切入点表达式，开发者可以指定在哪些方法、类或包上应用切面。**
4. **通知（Advice）**：
   1. **通知是切面在特定的连接点上执行的动作。**Spring AOP支持多种类型的通知：
      - **前置通知（Before Advice）**：在连接点之前执行。
      - **后置通知（After Advice）**：在连接点之后执行，无论方法是否抛出异常。
      - **返回通知（After Returning Advice）**：在连接点正常返回后执行。
      - **异常通知（After Throwing Advice）**：在方法抛出异常后执行。
      - **环绕通知（Around Advice）**：围绕连接点执行，可以在方法执行前后进行自定义的操作。
5. **目标对象（Target Object）**：
   1. 目标对象是被一个或多个切面切入的对象，也就是**切面最终作用的对象**。
6. **代理（Proxy）**：
   1. Spring AOP基于代理模式实现，代理对象是包含切面逻辑的对象，它包装了目标对象并在必要时将方法调用委托给目标对象。
7. **织入（Weaving）**：
   1. 织入是将切面应用到目标对象并创建代理对象的过程。Spring AOP在运行时动态织入切面。

### Spring AOP 的实现方式

Spring AOP通过代理模式实现，主要使用两种代理方式：

1. **JDK 动态代理**：
   1. 适用于基于接口的代理。如果目标对象实现了一个或多个接口，Spring AOP默认使用JDK动态代理为这些接口创建代理对象。
2. **CGLIB** **代理**：
   1. 适用于没有实现接口的类。CGLIB代理是基于子类的代理，Spring AOP会为目标类创建一个子类代理对象。

### 使用示例

以下是一个简单的Spring AOP示例，展示了如何使用AOP来记录方法的执行时间。

#### 1. 定义切面类

```Java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();

        Object proceed = joinPoint.proceed();

        long executionTime = System.currentTimeMillis() - startTime;

        System.out.println(joinPoint.getSignature() + " executed in " + executionTime + "ms");

        return proceed;
    }
}
```

#### 2. 配置Spring AOP

如果你使用的是Spring Boot，你不需要额外的配置，因为Spring Boot自动启用AOP。如果是传统Spring项目，可以在配置类中启用AOP：

```Java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy
public class AppConfig {
    // 配置类内容
}
```

#### 3. 目标对象

```Java
import org.springframework.stereotype.Service;

@Service
public class MyService {

    public void performTask() {
        // 业务逻辑
        System.out.println("Performing task...");
    }
}
```

#### 4. 测试AOP

在应用运行时，调用`MyService`中的`performTask`方法时，Spring AOP会自动在方法执行前后执行`LoggingAspect`中的环绕通知，记录方法的执行时间。

```Java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class AppRunner {

    @Autowired
    private MyService myService;

    public void run() {
        myService.performTask();
    }
}
```

### 实际应用场景

1. **日志记录**：自动记录方法的调用信息、执行时间等。
2. **事务管理**：自动管理数据库事务，简化事务处理代码。
3. **权限控制**：在方法调用前验证用户的权限。
4. **异常处理**：统一处理方法执行中的异常。

### 总结

Spring AOP 是一个强大的编程模型，可以帮助开发者将横切关注点从业务逻辑中分离出来，简化代码维护，并提高代码的可重用性和模块化程度。通过AOP，常见的系统级服务（如日志记录、事务管理）可以在不修改业务代码的情况下灵活地应用到多个对象上。

Coding不易，棒棒，加油！