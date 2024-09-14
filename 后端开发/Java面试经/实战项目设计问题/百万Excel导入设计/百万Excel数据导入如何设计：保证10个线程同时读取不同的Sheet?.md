

## 业务背景

业务方提供一份Excel，内部有20个左右的Sheet，每一份Sheet下是6万行的数据，大概是十几列，总计是120万行的数据，想要读取并存储到数据库中。

### 代码如何实现多线程读取不同 Sheet 的关键点：

1. **线程池**的使用：使用了 `ExecutorService` 来提交多个并行任务，每个任务负责处理一个 Sheet。

```Java
for (int i = 0; i < 20; i++) {
    int sheetNo = i;
    executorService.submit(() -> readAndProcessSheet(filePath, sheetNo));  // 每个线程处理不同的Sheet
}
```

通过循环，`sheetNo` 每次都不同，这就确保了每个线程处理的是不同的 Sheet。

2.**读取特定 Sheet**：

（1）`EasyExcel.read(filePath, DataEntity.class, new DataListener()).sheet(sheetNo).doRead();` 通过指定 `sheetNo` 参数，每个线程都只会读取特定的 Sheet。因为每个线程处理的 Sheet 不同，数据读取和处理的顺序不会被打乱。

（2）**每个线程处理独立的数据批次**：

在 `DataListener` 中，`dataBatch` 是每个线程独立的，因此不同线程不会互相干扰。demo示例，说明 10 个线程同时读取不同 Sheet：

```Java
public void importExcel(String filePath) {
    // 创建线程池，最大10个线程
    ExecutorService executorService = Executors.newFixedThreadPool(10);

    // 遍历每个Sheet，每个Sheet分配给一个线程处理
    for (int i = 0; i < 20; i++) {
        int sheetNo = i;  // 指定不同的Sheet编号
        executorService.submit(() -> readAndProcessSheet(filePath, sheetNo));
    }

    // 关闭线程池，确保所有线程完成任务
    executorService.shutdown();
    try {
        // 等待所有线程完成任务
        if (!executorService.awaitTermination(60, TimeUnit.MINUTES)) {
            executorService.shutdownNow();
        }
    } catch (InterruptedException e) {
        executorService.shutdownNow();
        Thread.currentThread().interrupt();
    }
}
```

总结：

- 通过 `ExecutorService` 创建线程池，每个线程并行处理一个 Sheet。
- `EasyExcel.read(...).sheet(sheetNo).doRead();` 确保每个线程只处理指定编号的 Sheet。
- 由于每个线程的操作是独立的，EasyExcel 的流式读取保证了读取顺序正确，且不会产生线程安全问题。

因此，上述代码可以保证 10 个线程同时读取不同的 Sheet，并且读取和处理过程都是线程安全且高效的。









