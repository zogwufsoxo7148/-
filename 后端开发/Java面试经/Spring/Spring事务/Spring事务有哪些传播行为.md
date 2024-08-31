Spring 事务的传播行为（Propagation Behavior）**定义了一个事务方法如何与现有的事务交互**。传播行为可以控制事务方法在被调用时是应该开启一个新的事务、加入到当前事务，还是以不同的方式进行处理。

### Spring 事务的传播行为有以下几种：

### 1. **REQUIRED**

- **描述**：**如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。**
- **适用场景**：这是最常用的传播行为，适用于大多数情况。

```Java
@Transactional(propagation = Propagation.REQUIRED)
public void perform() {
    // 业务逻辑
}
```

### 2. **REQUIRES_NEW**

- **描述**：总是创建一个新的事务。如果当前存在事务，则将其挂起。
- **适用场景**：当你需要强制启动一个新的事务，并且不希望与外部事务共享时使用。

```Java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void perform() {
    // 业务逻辑
}
```

### 3. **SUPPORTS**

- **描述**：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务方式执行。
- **适用场景**：适用于既可以在事务中执行，也可以在非事务中执行的操作。

```Java
@Transactional(propagation = Propagation.SUPPORTS)
public void perform() {
    // 业务逻辑
}
```

### 4. **NOT_SUPPORTED**

- **描述**：总是以非事务方式执行，如果当前存在事务，则将其挂起。
- **适用场景**：当你不希望在事务中执行某些操作时使用，例如一些不需要事务管理的操作。

```Java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void perform() {
    // 业务逻辑
}
```

### 5. **MANDATORY**

- **描述**：必须在一个现有事务中执行，如果当前没有事务，则抛出异常。
- **适用场景**：当方法必须在一个已有的事务中执行时使用，如果调用方未启动事务，抛出异常提醒。

```Java
@Transactional(propagation = Propagation.MANDATORY)
public void perform() {
    // 业务逻辑
}
```

### 6. **NEVER**

- **描述**：必须以非事务方式执行，如果当前存在事务，则抛出异常。
- **适用场景**：当你明确要求该方法不在事务中运行时使用，如果调用方有事务，则抛出异常。

```Java
@Transactional(propagation = Propagation.NEVER)
public void perform() {
    // 业务逻辑
}
```

### 7. **NESTED**

- **描述**：如果当前存在事务，则在当前事务中创建一个嵌套事务（即保存点）；如果当前没有事务，则与 `REQUIRED` 行为类似，创建一个新的事务。
- **适用场景**：当需要在一个大事务中执行某些子操作，并且希望这些子操作能够独立回滚时使用。

```Java
@Transactional(propagation = Propagation.NESTED)
public void perform() {
    // 业务逻辑
}
```

### 总结：

- **REQUIRED** 和 **REQUIRES_NEW** 是最常用的传播行为，前者适用于大多数情况，后者适用于需要独立事务的场景。
- **SUPPORTS** 和 **NOT_SUPPORTED** 适用于事务可选的情况。
- **MANDATORY** 和 **NEVER** 强制要求存在或不存在事务。
- **NESTED** 提供了在一个事务中管理子事务的功能，特别适合复杂的事务场景。

根据实际需求选择合适的传播行为，可以更好地控制事务的执行逻辑。