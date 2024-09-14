前端，在用户点击后，对按钮做置灰操作。但有些情况，用户会绕过置灰，实现重复点击。

后端，对客户端携带的token，验证是否使用过；验证逻辑，存储在数据库中，验证逻辑使用悲观锁或者乐观锁实现。

![img](https://zxgrz7s70kt.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmYwNjE1OTExZWY4NmUzMzQ4YjllZTkwYTlmY2U0MzNfZTF0a3dqODVPa2RvV2xGQVp3R0EzbE9abk8zMlV4U2VfVG9rZW46Rk9JVmJEa3RFb1VJN2Z4RjV1VWNJTXBqbjliXzE3MjU4NDk5Njc6MTcyNTg1MzU2N19WNA)

## **前端按钮置灰**

- **前端按钮置灰**：在用户点击按钮后，将按钮禁用一段时间或直到请求响应。
  - **优点**：简单易用，减少用户重复点击。
  - **缺点**：用户可以绕过前端限制，使用浏览器开发工具重新启用按钮，因此这种方式不能完全防止重复提交。

```HTML
<button id="submitBtn" onclick="submitForm()">提交</button>

<script>
function submitForm() {
    const submitBtn = document.getElementById('submitBtn');
    
    // 禁用按钮
    submitBtn.disabled = true;

    // 模拟表单提交操作
    setTimeout(() => {
        alert("Form Submitted!");
        // 重新启用按钮
        submitBtn.disabled = false;
    }, 2000); // 模拟请求响应时间
}
</script>
```

## **使用token机制（**本地缓存）

- **使用token机制**：用户在访问页面时获取一个token，提交操作时携带token，服务端验证token的有效性。这种方法通过数据库锁来确保操作的幂等性。
  - **优点**：防止用户重复提交请求，尤其是在多次点击按钮的情况下，保证请求的幂等性。
  - **缺点**：如果 Token 存储不当，可能会引入安全隐患，建议使用 Redis 等外部存储管理 Token 生命周期。

```Java
@RestController
public class OrderController {

    // Token存储 (可以使用Redis)
    private Map<String, Boolean> tokenStore = new ConcurrentHashMap<>();

    // 页面加载时，生成并返回一个token
    @GetMapping("/getToken")
    public String getToken() {
        String token = UUID.randomUUID().toString();
        tokenStore.put(token, true);
        return token;
    }

    // 提交表单时，验证token
    @PostMapping("/submitOrder")
    public ResponseEntity<String> submitOrder(@RequestParam String token) {
        // 验证token是否存在且有效
        if (!tokenStore.containsKey(token) || !tokenStore.get(token)) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Invalid or expired token");
        }

        // 一旦token验证通过，将其设置为无效
        tokenStore.put(token, false);

        // 处理订单逻辑
        return ResponseEntity.ok("Order submitted successfully");
    }
}
<form id="orderForm" method="POST" action="/submitOrder">
    <input type="hidden" id="token" name="token">
    <button type="submit">提交订单</button>
</form>

<script>
// 页面加载时获取token
fetch('/getToken')
    .then(response => response.text())
    .then(token => {
        document.getElementById('token').value = token;
    });
</script>
```

## **滑动窗口限流**

- 通过限制在一定时间内用户只能发起一次请求，来防止重复点击。这种方法适用于控制请求频率的场景。
  - **优点**：可以有效防止用户在短时间内频繁发起请求，避免重复提交。
  - **缺点**：滑动窗口限流比较适合频繁请求控制，不适合对特定操作的幂等控制。

```Java
@RestController
public class RateLimiterController {

    @Autowired
    private StringRedisTemplate redisTemplate;

    // 限制用户1分钟只能提交1次
    @PostMapping("/submitOrder")
    public ResponseEntity<String> submitOrder(@RequestParam String userId) {
        String key = "submit_rate_limit:" + userId;
        
        // 获取Redis中的限流信息
        String rateLimit = redisTemplate.opsForValue().get(key);
        
        if (rateLimit != null) {
            return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS).body("Too many requests. Please try later.");
        }

        // 如果通过限流，则设置1分钟的限流
        redisTemplate.opsForValue().set(key, "1", 60, TimeUnit.SECONDS);

        // 处理订单提交
        return ResponseEntity.ok("Order submitted successfully");
    }
}
```

## **布隆**过滤器

- **布隆过滤器**：使用布隆过滤器快速判断操作是否已执行过，如果未执行则进行操作，如果已执行则拒绝。这种方法适用于快速判断和减少数据库压力。
  - **优点**：布隆过滤器在大规模数据中效率很高，能够快速判断数据是否已经存在。
  - **缺点**：布隆过滤器有误判率，可能会误判一个没有提交过的订单为重复提交。

```Java
import io.rebloom.client.Client;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
public class OrderController {

    @Autowired
    private Client bloomFilter;

    // 假设布隆过滤器的名字是 "order_bloom"
    @PostMapping("/submitOrder")
    public ResponseEntity<String> submitOrder(@RequestParam String orderId) {
        // 检查布隆过滤器中是否已存在该订单号
        boolean exists = bloomFilter.exists("order_bloom", orderId);

        if (exists) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Duplicate order submission");
        }

        // 将新订单号添加到布隆过滤器
        bloomFilter.add("order_bloom", orderId);

        // 处理订单逻辑
        return ResponseEntity.ok("Order submitted successfully");
    }
}
```

## **数据库锁机制**

- **数据库锁机制**：参考某些框架的实现，将表单信息保存在数据库中，再次提交时进行校验，如果内容相同且时间间隔短则拒绝请求。这种方法通过数据库来保证操作的幂等性。
  - **优点**：数据库唯一约束可以彻底防止重复提交，即使多个并发请求都提交同一订单号，也只能保存一条记录
  - **缺点**：数据库锁机制可能会在高并发场景下带来一定的性能开销，不适合超高并发场景。

```Java
@Entity
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String orderId;

    private String userId;
    private Date createTime;

    // getter/setter
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    Optional<Order> findByOrderId(String orderId);
}

@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Transactional
    public String submitOrder(String orderId, String userId) {
        // 查找是否已有相同订单号的记录
        if (orderRepository.findByOrderId(orderId).isPresent()) {
            throw new IllegalArgumentException("Duplicate order submission");
        }

        // 创建新订单
        Order order = new Order();
        order.setOrderId(orderId);
        order.setUserId(userId);
        order.setCreateTime(new Date());

        orderRepository.save(order);

        return "Order submitted successfully";
    }
}

@RestController
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/submitOrder")
    public ResponseEntity<String> submitOrder(@RequestParam String orderId, @RequestParam String userId) {
        try {
            String result = orderService.submitOrder(orderId, userId);
            return ResponseEntity.ok(result);
        } catch (IllegalArgumentException e) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(e.getMessage());
        }
    }
}
```

## 数据库乐观锁

乐观锁和分布式锁可以有效地防止并发请求时的重复提交或数据更新冲突。

乐观锁通过在数据库中增加一个 `version` 字段来实现，每次更新数据时检查该字段的版本号，只有当版本号匹配时，才允许更新成功。

首先，在数据库的表中添加一个 `version` 字段：

```SQL
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id VARCHAR(255) UNIQUE,
    user_id VARCHAR(255),
    status VARCHAR(50),
    version INT DEFAULT 0  -- 版本号，作为乐观锁
);
```

### 实体类 `Order.java`：

```Java
import javax.persistence.*;

@Entity
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true)
    private String orderId;

    private String userId;

    private String status;

    @Version  // 乐观锁的版本号字段
    private Integer version;

    // Getters and Setters
}
```

### Repository 接口 `OrderRepository.java`：

```Java
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface OrderRepository extends JpaRepository<Order, Long> {
    Optional<Order> findByOrderId(String orderId);
}
```

### 服务层 `OrderService.java`：

```Java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Transactional
    public String updateOrderStatus(String orderId, String newStatus) {
        Order order = orderRepository.findByOrderId(orderId)
                .orElseThrow(() -> new IllegalArgumentException("Order not found"));

        // 更新订单状态，同时会检查版本号是否匹配
        order.setStatus(newStatus);
        orderRepository.save(order);

        return "Order status updated successfully";
    }
}
```

### 控制器 `OrderController.java`：

```Java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/updateOrderStatus")
    public ResponseEntity<String> updateOrderStatus(@RequestParam String orderId, @RequestParam String newStatus) {
        try {
            String result = orderService.updateOrderStatus(orderId, newStatus);
            return ResponseEntity.ok(result);
        } catch (Exception e) {
            return ResponseEntity.badRequest().body(e.getMessage());
        }
    }
}
```

说明：

- 每次更新 `Order` 时，Mybatis会自动检查 `version` 字段。如果在更新时发现 `version` 字段与数据库中的不一致（说明已经有其他请求更新了数据），则会抛出 `OptimisticLockException`。
- 乐观锁适用于读操作远多于写操作的场景，适合防止并发写冲突。

## 数据库悲观锁

使用 **数据库的**悲观锁 来防止用户重复点击，可以通过 **SQL** **锁机制** 来确保在处理当前操作时，阻止其他请求对同一条记录进行操作。悲观锁会在事务处理期间锁定一行数据，防止其他事务修改或读取该行数据，从而有效避免并发请求导致的数据冲突或重复提交。

在 **Spring Boot + MyBatis** 技术栈中，通常通过在 SQL 语句中使用 `SELECT ... FOR UPDATE` 实现悲观锁。`FOR UPDATE` 会锁定查询到的行，直到事务提交或回滚为止，其他事务在这段时间内无法对该行数据进行修改。

### 1. **创建订单表**

假设我们有一个订单表，包含订单 ID 和状态字段。

```SQL
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id VARCHAR(255) UNIQUE,
    user_id VARCHAR(255),
    status VARCHAR(50) DEFAULT 'PENDING'
);
```

### 2. **MyBatis Mapper 接口**

在 MyBatis 中，我们通过编写 SQL 来实现悲观锁。`FOR UPDATE` 语句用于锁定一条记录。

```Java
@Mapper
public interface OrderMapper {

    // 查询并锁定订单，防止重复操作
    @Select("SELECT * FROM orders WHERE order_id = #{orderId} FOR UPDATE")
    Order selectOrderForUpdate(String orderId);

    // 更新订单状态
    @Update("UPDATE orders SET status = #{status} WHERE order_id = #{orderId}")
    void updateOrderStatus(@Param("orderId") String orderId, @Param("status") String status);
}
```

### 3. **Service 层实现**悲观锁逻辑

在 Service 层中，我们可以通过事务管理和悲观锁来防止用户重复点击提交订单。

```Java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Transactional // 使用事务确保数据一致性
    public String processOrder(String orderId) {
        // 查询并锁定订单（悲观锁）
        Order order = orderMapper.selectOrderForUpdate(orderId);

        // 检查订单状态，防止重复操作
        if (!"PENDING".equals(order.getStatus())) {
            throw new IllegalStateException("Order has already been processed");
        }

        // 模拟订单处理逻辑
        try {
            Thread.sleep(2000); // 模拟处理时间
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // 更新订单状态
        orderMapper.updateOrderStatus(orderId, "COMPLETED");

        return "Order processed successfully";
    }
}
```

### 4. **Controller** **层调用**

在 Controller 中，调用 `OrderService` 处理订单。

```Java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/process")
    public ResponseEntity<String> processOrder(@RequestParam String orderId) {
        try {
            String result = orderService.processOrder(orderId);
            return ResponseEntity.ok(result);
        } catch (IllegalStateException e) {
            return ResponseEntity.status(HttpStatus.CONFLICT).body(e.getMessage());
        }
    }
}
```

关键点说明

1. **悲观锁** **(**`FOR UPDATE`)：在 `OrderMapper` 中，`SELECT ... FOR UPDATE` 会在数据库中锁定查询到的订单行，其他事务必须等待当前事务提交后才能访问该行数据。这就防止了多个用户同时点击按钮并发提交的情况。
2. **事务管理**：使用 `@Transactional` 注解确保所有操作在同一个事务中执行。如果操作失败，可以通过回滚事务来恢复原始状态。
3. **状态检查**：在悲观锁生效的情况下，仍然需要通过检查订单的状态字段（如 `status`）来防止重复提交。如果订单状态已经不是 `PENDING`，说明订单已经处理过了，可以直接拒绝请求。
4. **并发问题的解决**：通过悲观锁，确保同一时间只能有一个线程处理某个订单，其他并发请求会被阻塞直到锁释放。

运行流程

1. 用户点击按钮提交订单时，系统通过 `SELECT ... FOR UPDATE` 锁定对应订单。
2. 如果该订单尚未处理（状态为 `PENDING`），则继续处理订单逻辑，并更新状态为 `COMPLETED`。
3. 如果在处理过程中，另一个用户试图重复提交该订单，由于该订单行已被锁定，后续请求会被阻塞，直到前一个事务完成并释放锁。
4. 一旦事务提交，锁释放，系统会根据订单状态决定是否处理新请求或拒绝重复提交。

### 总结

通过使用悲观锁（`FOR UPDATE`）和事务机制，可以确保在处理订单时避免并发冲突，防止用户重复点击导致的订单重复处理问题。在 MyBatis 和 Spring Boot 组合中，这种方法非常有效。