## SpringBean一共有几种作用域？

在Spring中，Bean的作用域（Scope）定义了Bean的生命周期及其在容器中的可见性。Spring框架默认支持以下五种作用域：

### 1. Singleton（单例作用域）：

- **描述**：这是**Spring中的默认****作用域**。对于**一个给定的Bean定义，Spring容器中仅创建一个Bean实例，并在整个Spring容器中共享**。无论在何处注入该Bean，始终获得相同的实例。
- **使用场景**：适用于无状态的Bean，如服务类、DAO类等。
- **配置**：
  - 默认就是Singleton作用域，无需额外配置。
  - 可以通过注解显式指定：`@Scope("singleton")`

```Java
@Component
@Scope("singleton")
public class MySingletonBean {
    // 单例Bean逻辑
}
```

### 2. Prototype（原型作用域）：

- **描述**：在Prototype作用域下，**每次请求都会创建一个新的Bean实例**。这意味着，**每次注入、调用或获取该Bean时，Spring都会返回一个新的实例。**
- **使用场景**：适用于有状态的Bean，或者需要在每次使用时都创建新实例的场景。
- **配置**：
  - **使用****`@Scope("prototype")`****注解显式指定。**

```Java
@Component
@Scope("prototype")
public class MyPrototypeBean {
    // 原型Bean逻辑
}
```

### 3. Request（请求作用域）：

- **描述**：在Request作用域下，**每次HTTP请求都会创建一个新的Bean实例，并在该请求的整个生命周期内共享这个实例。**当请求结束后，该Bean实例会被销毁。此作用域主要用于Web应用程序。
- **使用场景**：适用于Web应用中的组件，如处理单个HTTP请求的控制器或服务。
- **配置**：
  - 使用`@Scope("request")`注解显式指定。

```Java
@Component
@Scope("request")
public class MyRequestScopedBean {
    // Request作用域Bean逻辑
}
```

### 4. Session（会话作用域）：

- **描述**：在Session作用域下，**每个HTTP会话都会创建一个Bean实例，并在该会话的整个生命周期内共享这个实例**。当会话结束后，该Bean实例会被销毁。此作用域也主要用于Web应用程序。
- **使用场景**：适用于Web应用中的组件，如需要在用户会话期间保持状态的对象。
- **配置**：
  - 使用`@Scope("session")`注解显式指定。

```Java
@Component
@Scope("session")
public class MySessionScopedBean {
    // Session作用域Bean逻辑
}
```

### 5. Application（应用程序作用域）：

- **描述**：在Application作用域下，**每个ServletContext（****Web应用****的上下文）会创建一个Bean实例，并在该ServletContext的整个生命周期内共享这个实例**。当ServletContext被销毁时，该Bean实例也会被销毁。
- **使用场景**：适用于需要在整个Web应用范围内共享的组件，如全局配置或资源管理器。
- **配置**：
  - 使用`@Scope("application")`注解显式指定。

```Java
@Component
@Scope("application")
public class MyApplicationScopedBean {
    // Application作用域Bean逻辑
}
```

### 总结

Spring框架支持五种主要的Bean作用域：

1. **Singleton**（默认作用域） - 全局单例
2. **Prototype** - 每次请求创建一个新实例
3. **Request** - 每个HTTP请求一个实例（Web应用）
4. **Session** - 每个HTTP会话一个实例（Web应用）
5. **Application** - 每个ServletContext一个实例（Web应用）