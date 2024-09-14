## 业务背景

业务方提供一份Excel，内部有20个左右的Sheet，每一份Sheet下是6万行的数据，大概是十几列，总计是120万行的数据，想要读取并存储到数据库中。

## 最佳实践Demo

仅供参考，根据自己实际的业务情况进行调整。

### 1. Maven 依赖（`pom.xml`）

```XML
<dependencies>
    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!-- Spring Boot MyBatis Starter -->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>2.1.4</version>
    </dependency>

    <!-- EasyExcel -->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>easyexcel</artifactId>
        <version>3.0.5</version>
    </dependency>

    <!-- HikariCP for database connection pool -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>3.4.5</version>
    </dependency>

    <!-- Logging -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-logging</artifactId>
    </dependency>

    <!-- Lombok for reducing boilerplate code -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.16</version>
        <scope>provided</scope>
    </dependency>

    <!-- Spring Boot Web Starter (for REST API) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

</dependencies>
```

### 2. 数据库配置（`application.yml`）

```YAML
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/your_db_name?useUnicode=true&characterEncoding=utf-8&useSSL=false
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 30  # 数据库连接池大小
      minimum-idle: 5        # 最小空闲连接数
      idle-timeout: 30000    # 空闲连接超时时间
      max-lifetime: 1800000  # 连接存活时间
      connection-timeout: 20000 # 连接超时时间
logging:
  level:
    com.example.demo: DEBUG  # 开启调试日志
```

### 3. 实体类（`DataEntity.java`）

```Java
import lombok.Data;
@Data
public class DataEntity {
    private Long id;
    private String column1;
    private String column2;
    // Other columns...
}
```

### 4. MyBatis Mapper 接口（`DataMapper.java`）

```Java
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

import java.util.List;

@Mapper
public interface DataMapper {

    // 批量插入方法
    void batchInsert(@Param("dataList") List<DataEntity> dataList);
}
```

### 5. 导入服务实现（`ExcelImportService.java`）

```Java
import com.alibaba.excel.EasyExcel;
import com.alibaba.excel.context.AnalysisContext;
import com.alibaba.excel.event.AnalysisEventListener;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;

@Slf4j
@Service
public class ExcelImportService {

    @Autowired
    private DataMapper dataMapper;

    @Autowired
    private ExecutorService executorService;  // 使用线程池

    private static final int BATCH_SIZE = 1000;  // 批次大小

    public void importExcel(String filePath) {
        // 遍历每个Sheet，每个Sheet分配给一个线程处理
        for (int i = 0; i < 20; i++) {
            int sheetNo = i;
            executorService.submit(() -> readAndProcessSheet(filePath, sheetNo));
        }
        
        // 关闭线程池，不再接受新的任务，等待所有线程完成
        executorService.shutdown();
        try {
            // 等待线程池中的所有任务完成，设定超时时间（例如60分钟）
            if (!executorService.awaitTermination(60, TimeUnit.MINUTES)) {
                // 如果超时未完成任务，强制关闭
                executorService.shutdownNow();
            }
        } catch (InterruptedException e) {
            // 在等待期间被中断时，强制关闭线程池
            executorService.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    public void readAndProcessSheet(String filePath, int sheetNo) {
        // 使用 EasyExcel 流式读取每个 Sheet
        EasyExcel.read(filePath, DataEntity.class, new DataListener()).sheet(sheetNo).doRead();
    }

    // 自定义 EasyExcel 的监听器来处理每批次数据
    @Slf4j
    public class DataListener extends AnalysisEventListener<DataEntity> {

        private List<DataEntity> dataBatch = new ArrayList<>();  // 缓存当前批次数据
        
        //EasyExcel的流式读取机制，保证每一次读取都会从上次结束的地方开始
        @Override
        public void invoke(DataEntity data, AnalysisContext context) {
            dataBatch.add(data);

            // 如果缓存的数据达到批次大小，开始处理数据
            if (dataBatch.size() >= BATCH_SIZE) {
                processBatch();
                dataBatch.clear();
            }
        }

        @Override
        public void doAfterAllAnalysed(AnalysisContext context) {
            // 处理剩余数据
            if (!dataBatch.isEmpty()) {
                processBatch();
            }
        }

        // 处理批次数据，带有事务控制
        @Transactional(rollbackFor = Exception.class)
        public void processBatch() {
            try {
                log.debug("Processing batch of size: {}", dataBatch.size());
                dataMapper.batchInsert(dataBatch);
            } catch (Exception e) {
                log.error("Failed to insert batch", e);
                throw new RuntimeException("Batch insert failed", e);
            }
        }
    }
}
```

### 6. 线程池配置（`ThreadPoolConfig.java`）

```Java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
public class ThreadPoolConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(6);  // 核心线程数
        executor.setMaxPoolSize(10);  // 最大线程数
        executor.setQueueCapacity(500);  // 队列容量
        executor.setThreadNamePrefix("ExcelImport-");
        executor.initialize();
        return executor;
    }
}
```

### 7. 控制器（`ExcelImportController.java`）

```Java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/excel")
public class ExcelImportController {

    @Autowired
    private ExcelImportService excelImportService;

    // 启动Excel导入任务
    @GetMapping("/import")
    public String importExcel() {
        String filePath = "/path/to/your/excel/file.xlsx";
        excelImportService.importExcel(filePath);
        return "Import started";
    }
}
```

### 8. 日志和异常处理

- **日志记录**：每次批量插入时，记录批次日志和插入结果。
- **异常处理**：批量插入时，如果插入失败，捕获异常并回滚事务，同时记录错误日志。

### 9. JVM 和性能调优

根据服务器的 CPU 和内存配置，调整 JVM 参数，确保能够处理大批量数据：

```Bash
JAVA_OPTS="-Xms1024m -Xmx4096m -XX:MaxMetaspaceSize=512m -XX:+UseG1GC"
```

### 10. 总结

这个示例代码实现了一个基本的大规模 Excel 导入系统，关键点包括：

- **多线程**并发处理多个 Sheet，提高了导入的性能。
- **批量插入和事务控制**，确保每批数据的原子性和一致性。
- **日志记录**和异常处理，便于后续调试和错误排查。

根据实际业务需求，还可以进一步扩展，如增加权限校验、导入进度反馈等功能。