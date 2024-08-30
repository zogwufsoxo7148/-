

### 1.说说你对ThreadLocal的理解

```Java
在并发编程中，当多个线程同时对共享资源进行访问，在特殊的情况下会造成数据安全问题。

解决方式，比如使用synchronized、lock、ReentrantLock这样的锁机制，但是使用锁，就会带来性能的损耗。ThreadLocal就解决这些问题。

ThreadLocal为每个线程开辟独有的本地变量，避免了数据的不安全问题。
```

### 2.项目中怎么应用ThreadLocal？

- 使用锁机制的并发情况，只能记录总数，但是无法记录每一个线程具体做了多少。

```Java
//资源类，保时米SU7轿跑,本门店有3个销售(3个线程)
class SU7
{
    @Getter
    private int saleTotal;//本门店总体销售额
    public synchronized void saleTotal()
    {
        saleTotal++;
    }
}

/**
 * ThreadLocal案例演示
 *
 * 第一次需求：
 *  3个销售卖保时米SU7，求3个销售本团队总体销售额
 *
 * @auther zzyy
 * @create 2024-05-06 11:16
 */
public class ThreadLocalDemoV1
{
    public static void main(String[] args) throws InterruptedException
    {
        // 线程   操作  资源类
        SU7 su7 = new SU7();
        CountDownLatch countDownLatch = new CountDownLatch(3);

        for (int i = 1; i <=3; i++)
        {
            new Thread(() -> {
                try {
                    for (int j = 1; j <= new Random().nextInt(3)+1 ; j++) {
                        su7.saleTotal();//本门店销售总和统计全部加
                    }
                    System.out.println(Thread.currentThread().getName()+"\t"+" 号销售卖出: "+su7.salePersonal.get());
                } finally {
                    countDownLatch.countDown();
                }
            },String.valueOf(i)).start();
        }
        countDownLatch.await();
        System.out.println(Thread.currentThread().getName()+"\t"+"销售总额: "+su7.getSaleTotal());
    }
}
```

- 使用ThreadLocal的并发情况，支持记录每一个线程具体完成了多少任务量

```Java
//资源类，保时米SU7轿跑,本门店有3个销售(3个线程)
class SU7
{
    @Getter
    private int saleTotal;//本门店总体销售额
    public synchronized void saleTotal()
    {
        saleTotal++;
    }

    //ThreadLocal 求3个销售的各自独立的个体销售额，不参加总和计算

    //不推荐使用版本：初始化
    /*ThreadLocal<Integer> salePersonal = new ThreadLocal<>(){
        @Override
        protected Integer initialValue()
        {
            return 0;
        }
    };*/

    //推荐使用版本：初始化
    ThreadLocal<Integer> salePersonal = ThreadLocal.withInitial(() -> 0);
    //每调用一次，累加一次
    public void salePersonal()
    {
        salePersonal.set(1 + salePersonal.get());
    }

}

/**
 * ThreadLocal案例演示
 *
 * 第一次需求：
 *  3个销售卖保时米SU7，求3个销售本团队总体销售额
 *
 * 第二次需求：
 * 3个销售卖保时米SU7，求3个销售的各自独立的个体销售额，不参加总和计算，
 *   希望各自分灶吃饭，各凭销售本事提成，按照出单数各自统计，
 *   每个销售都有自己的销售额指标，自己专属自己的，不和别人掺和
 *
 * @auther zzyy
 * @create 2024-05-06 11:16
 */
public class ThreadLocalDemoV1
{
    public static void main(String[] args) throws InterruptedException
    {
        // 线程   操作  资源类
        SU7 su7 = new SU7();
        CountDownLatch countDownLatch = new CountDownLatch(3);

        for (int i = 1; i <=3; i++)
        {
            new Thread(() -> {
                try {
                    for (int j = 1; j <= new Random().nextInt(3)+1 ; j++) {
                        su7.saleTotal();//本门店销售总和统计全部加
                        su7.salePersonal();//各个销售独立的销售额，之和自己有关
                    }
                    System.out.println(Thread.currentThread().getName()+"\t"+" 号销售卖出: "+su7.salePersonal.get());
                } finally {
                    countDownLatch.countDown();
                    //su7.salePersonal.remove(); 只要使用ThreadLocal，一定要记得remove，否则内存泄漏
                }
            },String.valueOf(i)).start();
        }
        countDownLatch.await();
        System.out.println(Thread.currentThread().getName()+"\t"+"销售总额: "+su7.getSaleTotal());
    }
}
```
