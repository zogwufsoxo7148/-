使用Redis实现朋友圈点赞功能，包括记录点赞数量、支持查看、点赞和取消点赞操作，以及查看点赞顺序。通过使用ZSet数据结构，可以有效地存储点赞用户ID和点赞时间戳，实现点赞功能。

- **功能需求分析**: 朋友圈点赞功能需记录点赞数量、支持查看、点赞和取消点赞操作，并能查看点赞顺序。
- **数据结构选择**: 使用Redis的ZSet数据结构存储点赞信息，其中value为点赞用户ID，score为点赞时间的时间戳。
- **点赞操作实现**: 将用户ID和当前时间戳添加到ZSet中，如果用户已点赞，则更新其时间戳。
- **取消点赞操作实现**: 从ZSet中删除用户的ID。
- **查询点赞信息**: 使用ZREVRANGEBYSCORE命令逆序返回点赞用户的ID。
- **代码实现**: 提供了Java代码示例，包括点赞(likePost)、取消点赞(unlikePost)和查看点赞列表(getLikes)的方法。

我们使用 Spring Boot + Redisson + MyBatis 来实现朋友圈点赞功能，其中包括点赞、取消点赞、查看点赞列表和点赞顺序。使用 Redis 的 ZSet 数据结构存储点赞信息，`value` 为用户 ID，`score` 为点赞时间的时间戳。

### 1. **依赖配置**

首先，在 `pom.xml` 中添加依赖：

```XML
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!-- MyBatis Starter -->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>2.2.0</version>
    </dependency>

    <!-- Redisson Redis Client -->
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson-spring-boot-starter</artifactId>
        <version>3.16.6</version>
    </dependency>

    <!-- MySQL Connector -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 2. **Redis 配置**

在 `application.yml` 中配置 Redis 连接信息：

```YAML
spring:
  redis:
    host: localhost
    port: 6379
```

### 3. **MyBatis 数据库表**

在 MyBatis 中创建一张 `post` 表，用于存储朋友圈的帖子信息。数据库只用于存储帖子基本信息，点赞功能全部通过 Redis 实现。

```SQL
CREATE TABLE post (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    content TEXT NOT NULL,
    user_id BIGINT NOT NULL
);
```

### 4. **MyBatis Mapper**

定义 `PostMapper` 用于操作 `post` 表。该表只存储帖子相关信息。

```Java
@Mapper
public interface PostMapper {

    @Select("SELECT * FROM post WHERE id = #{postId}")
    Post selectPostById(@Param("postId") Long postId);

    @Insert("INSERT INTO post (content, user_id) VALUES (#{content}, #{userId})")
    @Options(useGeneratedKeys = true, keyProperty = "id")
    int insertPost(Post post);
}
```

### 5. **Redisson** **Service**

接下来，我们通过 Redisson 操作 Redis 的 ZSet 结构来实现点赞功能。我们将每个帖子的点赞信息存储在 Redis 中，Key 命名为 `post:like:{postId}`，其中 `postId` 是具体的帖子 ID，`value` 是用户 ID，`score` 是点赞的时间戳。

```Java
import org.redisson.api.RScoredSortedSet;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.List;
import java.util.Set;

@Service
public class LikeService {

    @Autowired
    private RedissonClient redissonClient;

    // 点赞操作
    public void likePost(Long postId, Long userId) {
        String key = "post:like:" + postId;
        RScoredSortedSet<Long> likeSet = redissonClient.getScoredSortedSet(key);
        // 使用当前时间戳作为评分，记录用户ID
        likeSet.add(Instant.now().getEpochSecond(), userId);
    }

    // 取消点赞操作
    public void unlikePost(Long postId, Long userId) {
        String key = "post:like:" + postId;
        RScoredSortedSet<Long> likeSet = redissonClient.getScoredSortedSet(key);
        likeSet.remove(userId);
    }

    // 获取点赞用户ID列表（按时间顺序）
    public Set<Long> getLikes(Long postId) {
        String key = "post:like:" + postId;
        RScoredSortedSet<Long> likeSet = redissonClient.getScoredSortedSet(key);
        // 按时间顺序返回用户ID列表
        return likeSet.readAll();
    }

    // 获取点赞用户ID列表（按时间逆序）
    public List<Long> getLikesByTimeDesc(Long postId) {
        String key = "post:like:" + postId;
        RScoredSortedSet<Long> likeSet = redissonClient.getScoredSortedSet(key);
        // 逆序返回点赞用户ID
        return likeSet.valueRangeReversed(0, -1);
    }

    // 获取点赞数量
    public int getLikeCount(Long postId) {
        String key = "post:like:" + postId;
        RScoredSortedSet<Long> likeSet = redissonClient.getScoredSortedSet(key);
        return likeSet.size();
    }
}
```

### 6. **Post Service**

实现对 `Post` 表的操作，创建帖子以及获取帖子信息。

```Java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class PostService {

    @Autowired
    private PostMapper postMapper;

    public Post createPost(String content, Long userId) {
        Post post = new Post();
        post.setContent(content);
        post.setUserId(userId);
        postMapper.insertPost(post);
        return post;
    }

    public Post getPost(Long postId) {
        return postMapper.selectPostById(postId);
    }
}
```

### 7. **Controller** **实现**

定义 `PostController` 来处理前端请求，包括创建帖子、点赞、取消点赞、获取点赞列表等操作。

```Java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Set;

@RestController
@RequestMapping("/posts")
public class PostController {

    @Autowired
    private PostService postService;

    @Autowired
    private LikeService likeService;

    // 创建帖子
    @PostMapping("/create")
    public ResponseEntity<Post> createPost(@RequestParam String content, @RequestParam Long userId) {
        Post post = postService.createPost(content, userId);
        return ResponseEntity.ok(post);
    }

    // 点赞
    @PostMapping("/like")
    public ResponseEntity<String> likePost(@RequestParam Long postId, @RequestParam Long userId) {
        likeService.likePost(postId, userId);
        return ResponseEntity.ok("Post liked successfully");
    }

    // 取消点赞
    @PostMapping("/unlike")
    public ResponseEntity<String> unlikePost(@RequestParam Long postId, @RequestParam Long userId) {
        likeService.unlikePost(postId, userId);
        return ResponseEntity.ok("Post unliked successfully");
    }

    // 获取点赞用户列表（按时间顺序）
    @GetMapping("/likes/{postId}")
    public ResponseEntity<Set<Long>> getLikes(@PathVariable Long postId) {
        Set<Long> likes = likeService.getLikes(postId);
        return ResponseEntity.ok(likes);
    }

    // 获取点赞用户列表（按时间逆序）
    @GetMapping("/likesDesc/{postId}")
    public ResponseEntity<List<Long>> getLikesByTimeDesc(@PathVariable Long postId) {
        List<Long> likes = likeService.getLikesByTimeDesc(postId);
        return ResponseEntity.ok(likes);
    }

    // 获取点赞数量
    @GetMapping("/likeCount/{postId}")
    public ResponseEntity<Integer> getLikeCount(@PathVariable Long postId) {
        int likeCount = likeService.getLikeCount(postId);
        return ResponseEntity.ok(likeCount);
    }
}
```

### 8. 实体类 **`Post`**

```Java
public class Post {

    private Long id;
    private String content;
    private Long userId;

    // Getters and setters
}
```

### 9. **Redisson** **配置**

在 `application.yml` 中配置 Redisson 的 Redis 连接：

```YAML
redisson:
  single-server-config:
    address: "redis://localhost:6379"
```

### 总结

通过 Redis 的 `ZSet` 数据结构，我们可以高效地实现朋友圈的点赞功能，包括点赞、取消点赞、查看点赞顺序等操作。`ZSet` 的 `value` 是用户 ID，`score` 是点赞时间戳，能够实现按时间顺序排序的点赞列表。

## Redis双写

在设计数据模型时，将点赞详情存储到 Redis 中而没有直接存储到数据库中，通常是出于性能、响应速度和并发处理的考虑。但存在数据丢失的风险。让我们来分析一下为什么采用 Redis 存储点赞详情的设计，以及如何应对 Redis 数据丢失的情况。

### 1. **为什么使用 Redis 存储点赞详情？**

Redis 是一种内存存储，具有极高的读写性能，适合处理高并发、实时性的场景。下面是选择 Redis 存储点赞详情的几个主要原因：

- **高并发性能**：点赞操作通常发生在高并发场景下（如热门帖子、大量用户同时点赞）。Redis 可以每秒处理数十万的请求，读写性能远高于传统的关系型数据库。相比之下，频繁的数据库写入操作可能会造成较大的压力。
- **快速响应**：用户点赞、取消点赞以及查询点赞数、点赞顺序等操作需要非常快速的响应时间。Redis 基于内存的特性可以在微秒级返回结果，确保用户体验良好。
- **排序操作**：Redis 的 `ZSet` 数据结构特别适合处理需要按时间排序的场景（比如按时间查看点赞顺序）。在关系型数据库中，频繁地对点赞数据进行排序查询可能会导致性能瓶颈。

### 2. **Redis 中存储点赞详情的缺点**

尽管 Redis 的性能非常高，但它是内存型数据库，这带来以下几个风险：

- **数据丢失风险**：Redis 中的数据在系统宕机、崩溃或者没有持久化的情况下可能会丢失。尤其是当 Redis 没有开启持久化（如 RDB 或 AOF）时，点赞数据无法在内存中恢复。
- **有限存储容量**：Redis 是内存型数据库，因此相比于硬盘存储的关系型数据库，内存容量较小，无法长时间保存大量的点赞数据。

### 3. **应对 Redis 数据丢失的解决方案**

#### 3.1 Redis 数据持久化

Redis 支持持久化机制，通过 RDB 或 AOF 的方式将数据写入磁盘，防止意外宕机或重启后数据丢失。

- **RDB** **持久化**：Redis 会定期将数据快照保存到磁盘中，定期生成 RDB 文件。可以通过配置保存策略，如 `save 900 1`（900秒内至少有1次修改）来设置保存频率。
- **AOF**（Append-Only File）持久化**：通过记录每次写操作日志，将操作按顺序追加到日志文件中，Redis 可以在重启时重放这些操作日志，恢复数据。

推荐使用 **AOF** 持久化，因为它更能保证数据的完整性，虽然带来一定的性能开销。

**Redis 配置持久化示例**：

```YAML
save 900 1        # 900秒内至少有1次数据修改，保存RDB快照
appendonly yes    # 开启AOF日志
```

#### 3.2 数据库的定期同步（双写策略）

可以设计一个机制，将 Redis 中的点赞数据定期同步到关系型数据库，以此作为备份方案。通过这种策略，即使 Redis 数据丢失，也可以通过数据库进行数据恢复。

- 定期将 Redis 中的点赞数据批量持久化到数据库中。例如，每小时执行一次同步，将 Redis 中的点赞数据写入数据库表。
- 通过消息队列（如 Kafka、RocketMQ）将点赞、取消点赞操作发送到异步任务中，任务会将这些操作同时写入 Redis 和数据库。

最佳案例：

1. 在点赞操作时，Redis 和数据库同时写入点赞数据。

```Java
public void likePost(Long postId, Long userId) {
    // 1. Redis 中记录点赞
    String key = "post:like:" + postId;
    RScoredSortedSet<Long> likeSet = redissonClient.getScoredSortedSet(key);
    likeSet.add(Instant.now().getEpochSecond(), userId);

    // 2. 数据库中记录点赞
    postLikeMapper.insertLike(postId, userId, Instant.now().getEpochSecond());
}
```

1. 定期同步 Redis 数据到数据库。

```Java
public void syncLikesToDatabase() {
    // 从 Redis 中获取所有点赞数据
    for (Post post : allPosts) {
        Set<Long> likes = likeService.getLikes(post.getId());
        for (Long userId : likes) {
            postLikeMapper.insertLike(post.getId(), userId, Instant.now().getEpochSecond());
        }
    }
}
```

#### 3.3 异常恢复机制

设计一种恢复机制，确保 Redis 数据丢失时，可以从数据库恢复数据。可以定期从数据库读取点赞记录，并将这些记录重新加载到 Redis 中。

**恢复 Redis 数据的示例代码**：

```Java
public void restoreLikesFromDatabase(Long postId) {
    // 查询数据库中该帖子的所有点赞记录
    List<PostLike> likes = postLikeMapper.selectLikesByPostId(postId);

    // 恢复到 Redis 中
    String key = "post:like:" + postId;
    RScoredSortedSet<Long> likeSet = redissonClient.getScoredSortedSet(key);
    for (PostLike like : likes) {
        likeSet.add(like.getTimestamp(), like.getUserId());
    }
}
```

### 4. **使用 Redis 的设计原则**

通常情况下，Redis 的点赞功能是基于以下原则设计的：

1. **高频操作用 Redis，低频操作用数据库**：Redis 适合处理大量频繁的点赞、取消点赞操作，而数据库则可以保存历史数据和做持久化备份。
2. **避免频繁写入数据库**：由于数据库的写入性能相对较慢，可以通过 Redis 缓存点赞数据，并在特定时间点或在点赞达到一定阈值后，再将数据批量同步到数据库中，减少数据库的压力。

### 5. **总结与建议**

- **Redis 的优势**：使用 Redis 实现点赞功能可以大大提升性能，特别是在高并发场景下能够减少数据库的压力并加快响应速度。
- **数据丢失问题**：由于 Redis 的数据存储在内存中，确实存在数据丢失的可能性，但可以通过 Redis 持久化（如 RDB、AOF）来降低数据丢失的风险。
- **多级存储方案**：为了确保数据安全，建议采用 Redis 存储和数据库双写的策略，Redis 负责高并发场景下的快速响应，数据库作为持久化存储备份。这样即便 Redis 数据丢失，仍然可以从数据库中恢复数据。

通过结合 Redis 的性能优势和数据库的持久化特性，可以设计一个既高效又可靠的点赞功能。