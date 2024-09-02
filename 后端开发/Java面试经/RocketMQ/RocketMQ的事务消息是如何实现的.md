今天，Java已经卷到“屎上雕花”的程度，八股文的准备如果仅仅靠背诵，很容易陷入“背了忘，忘了背”的死循环中。

所以，我们必须：结合具体的代码demo，尝试系统地掌握，才能更好的卷出一条活路。

RocketMQ 的事务消息用于实现分布式事务，使得跨多个服务或系统的事务操作能够保持一致性。RocketMQ 的事务消息实现遵循 “两阶段提交协议”，包括消息的预发送、事务执行和最终确认三个步骤。以下是 RocketMQ 事务消息的实现流程：

### 1. **发送半消息（Prepare Message）**

- **步骤**：生产者首先发送一条“半消息”到 Broker，这条消息是暂时不可被消费者消费的，因为事务还未提交。
- **作用**：Broker 收到半消息后，存储该消息但不立即将其投递给消费者。此时消息处于“待定状态”。

```Java
SendResult sendResult = producer.sendMessageInTransaction(message, localTransactionExecutor, arg);
```

### 2. **执行本地事务**

- **步骤**：生产者发送完半消息后，开始执行本地事务操作（例如，更新数据库）。
- **结果**：本地事务的执行结果决定了这条消息的最终状态——提交还是回滚。

```Java
@Transactional
public LocalTransactionState executeLocalTransaction(Message message, Object arg) {
    // 执行本地事务（如数据库操作）
    boolean success = performLocalTransaction();
    if (success) {
        return LocalTransactionState.COMMIT_MESSAGE;
    } else {
        return LocalTransactionState.ROLLBACK_MESSAGE;
    }
}
```

### 3. **提交或回滚事务**

- **提交事务**：如果本地事务执行成功，生产者提交这条事务消息，Broker 将消息标记为可投递状态，并将其投递给消费者。
- **回滚事务**：如果本地事务执行失败，生产者回滚这条事务消息，Broker 将删除该消息或标记为不可投递。

```Java
if (transactionStatus == LocalTransactionState.COMMIT_MESSAGE) {
    // 提交事务，消息将被消费者消费
} else if (transactionStatus == LocalTransactionState.ROLLBACK_MESSAGE) {
    // 回滚事务，消息将被丢弃
}
```

### 4. **事务状态回查**

- **场景**：如果 Broker 长时间没有收到生产者的事务确认消息（提交或回滚），它会主动向生产者发起回查，询问这条事务消息的最终状态。
- **回查实现**：生产者实现一个回查接口，Broker 会调用该接口获取本地事务的状态，并根据返回值决定是提交还是回滚事务。

```Java
@Override
public LocalTransactionState checkLocalTransaction(MessageExt message) {
    // 检查本地事务的状态
    boolean success = checkTransactionStatus();
    if (success) {
        return LocalTransactionState.COMMIT_MESSAGE;
    } else {
        return LocalTransactionState.ROLLBACK_MESSAGE;
    }
}
```

### RocketMQ 事务消息流程图（简化版）

```Plain
 +--------------------+       +-----------------+       +-------------------+
 |     Producer       |       |    Broker       |       |     Consumer      |
 +--------------------+       +-----------------+       +-------------------+
       |                           |                           |
       |--1. 发送半消息 (Prepare)--> |                           |
       |                           | 存储半消息                  |
       |                           |                           |
       |--2. 执行本地事务----------> |                           |
       |   (成功或失败)             |                            |
       |                           |                           |
       |--3. 提交或回滚事务状态----> |                            |
       |                           | 提交或删除消息              |
       |                           |                           |
       |                           | --4. 投递消息-->           |
       |                           |                           |
```

### 事务消息的几个关键点：

- **幂等性**：由于事务消息的回查机制，生产者可能会重复提交消息，因此确保本地事务操作的幂等性非常重要。
- **回查机制**：Broker 通过回查机制解决了网络抖动或生产者崩溃等导致事务状态不明的问题，确保事务的一致性。
- **事务状态**：本地事务的状态通过 `LocalTransactionState` 返回，包括 `COMMIT_MESSAGE`、`ROLLBACK_MESSAGE` 和 `UNKNOW` 三种状态，分别表示提交、回滚和未知状态。

### 总结：

RocketMQ 事务消息通过两阶段提交协议和事务状态回查机制，确保了分布式环境下事务的一致性。生产者发送半消息后执行本地事务，根据事务结果提交或回滚消息。如果 Broker 未能及时获取事务状态，它将回查生产者以确保消息最终的处理状态。

这里有一个问题，为什么要使用事务消息呢？

假如没有事务消息，本地事务执行成功，发送消息，但不巧的是：消息发送失败了。这个时候，提供者 和 消费方就会存在数据不一致的情况。

但假如，我们使用事务消息，提供者先发送一个“半消息”，半消息作为一个检测机制。如果本地事务成功，则commit，消费者可以继续消费。如果本地事务失败，则rollback，消息者和提供者数据依旧可以保持一致。

总结一句话：事务消息相当于一个检测机制，用来保证：最终的数据一致性。

Coding不易，棒棒，加油！