### 业务背景

业务方提供一份Excel，内部有20个左右的Sheet，每一份Sheet下是6万行的数据，大概是十几列，总计是120万行的数据，想要读取并存储到数据库中。

## 一个线程读取一个Sheet中的前1000条数据，怎么保证后续可以从10001条读取呢？

1. **EasyExcel 的流式读取机制**：
   1. EasyExcel 是基于事件驱动的流式读取工具，文件被顺序读取，`invoke()` 方法会在每次读取到一条数据时被触发。
   2. 每次调用 `invoke()` 方法时，数据会顺序被读取和处理。因此，1000 条数据读取完成后，接下来 EasyExcel 会自动读取第 1001 条数据，并且继续按照顺序触发 `invoke()` 方法，直到完成整个 Sheet 的读取。
2. **批量处理逻辑**：
   1. `dataBatch` 是一个缓冲列表，用来存储当前批次的 1000 条数据。
   2. 当 `dataBatch.size() >= BATCH_SIZE`（即达到 1000 条）时，`processBatch()` 方法被调用，这时当前批次的 1000 条数据会被处理并插入到数据库中，然后调用 `dataBatch.clear()` 清空批次缓存列表，准备读取和存储下一批数据。
   3. 因为 `dataBatch.clear()` 清除了前一批的缓存，所以不会重复处理已经处理过的数据，下一次调用 `invoke()` 时会接着从第 1001 条数据开始填充 `dataBatch`。
3. **顺序处理**：
   1. EasyExcel 的 `AnalysisEventListener` 保证了 `invoke()` 方法会按顺序触发。由于你是流式读取，每次批量处理 1000 条数据之后，接下来读取的数据会继续从上一批的末尾开始（即 1001 到 2000 条，依次类推），直到所有数据读取完毕。

关键点：

- **EasyExcel 的流式读取特性**：它不会一次性将整个文件加载到内存中，而是按行顺序读取，因此不需要担心读取顺序被打乱。
- **批次处理机制**：在达到批次大小（1000 条）时，会立即处理该批次并清空缓存，确保接下来从正确的位置继续读取。

关键导入Demo：

```java
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

因此，基于代码中的逻辑，当一个线程读取完前 1000 条数据并处理完后，系统会自动继续读取从第 1001 条开始的下一个批次，并按顺序处理数据。