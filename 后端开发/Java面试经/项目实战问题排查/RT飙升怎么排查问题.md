## 一、问题现象

在一个学生健康教育系统中，当大量新的学生健康数据（大概16万条）被导入到成绩管理系统时，由于数据量的激增，导致学生健康查询接口的响应时间（RT）显著增加。

## 二、问题原因

经过排查，发现问题是由于现有的数据库索引在处理大量新增数据时效率低下。为了解决这个问题，开发团队优化了数据库索引，创建了新的、更高效的索引，从而显著提高了成绩查询接口的响应速度。

## 三、排查流程

我们产看系统数据，RT接口明显增加的时间 和 大量数据导入的时间，刚好吻合；很明显，很大概率，这次RT飙高就是因为：大量数据存储导致的。

![img](https://zxgrz7s70kt.feishu.cn/space/api/box/stream/download/asynccode/?code=YWQxNGMwMmQwNWRmZjA1YjFhZmI2MDFlZDQyOGI2YWZfTkFGczNpdmxXeU9ycUNYOFNMN3BsUGlLM0QzdUc5Mm9fVG9rZW46U1NFWGJqc0lTb0x0VjV4a2cxSWM0ZnR2bm9jXzE3MjUyNzQ2NTY6MTcyNTI3ODI1Nl9WNA)

在服务器上下载并使用Arthas：trace + 目标类方法的路径 + -n 查询深度，发现：方法总耗时260ms，但有220ms都耗费在一个方法上。

确定了具体耗时的方法，继续使用Arthas的watch方法，查看耗时比较长的接口的入参情况，尝试找到规律。

但是我们的这个服务应用接入了内部开发的一个监控平台，可以看到接口的请求情况和耗时情况，最终层层追查确定了是：SQL语句的问题。

我们的查询中，使用了studentId、groupId做了联合索引，虽然索引生效了，但是：依旧还有上万的数据需要通过teacher_id和doctor_id来进行过滤。所以，添加了针对：teacher_id和doctor_id进行联合索引处理，后续的测试中，平均RT确实降低了很多。

![img](https://zxgrz7s70kt.feishu.cn/space/api/box/stream/download/asynccode/?code=NzQ1NjViMWQzOGE0NzI4YTAwN2U2YWJkNjgxZGEyMjVfSnk5amRYU3NyNFpMSTZtczZSZDd2VndIZGZobEFZNkZfVG9rZW46WDBXT2JlcEFkb0c4a1Z4TzBJaWNWZGZsbkNjXzE3MjUyNzQ2NTY6MTcyNTI3ODI1Nl9WNA)

这次问题的解决，主要是：依赖Arthas可以快速定位，进一步确定了是索引区分度不高的问题，最终通过进一步细分索引的区分度，降低了平均的RT。

## 四、详细参数

### trace命令的使用

- 查看指定方法的调用次数、平均耗时信息：trace com.example.MyClass MyMethod -n 1 --stat
- trace com.example.MyClass MyMethod -n 4：这条命令，监控下方的方法。

```Java
package com.example;

public class MyClass {

    public void MyMethod() {
        System.out.println("MyMethod called");
        level1();
    }

    private void level1() {
        System.out.println("Level 1");
        level2();
    }

    private void level2() {
        System.out.println("Level 2");
        level3();
    }

    private void level3() {
        System.out.println("Level 3");
        level4();
    }

    private void level4() {
        System.out.println("Level 4");
        try {
            Thread.sleep(200); // Simulate some processing time
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        MyClass myClass = new MyClass();
        myClass.MyMethod();
    }
}
```

命令具体监控情况：

```Java
`---ts=2024-09-02 10:00:00;thread_name=main;id=1;is_daemon=false;priority=5;TCCL=sun.misc.Launcher$AppClassLoader@18b4aac2
    `---[0.435279ms] com.example.MyClass:MyMethod()
        `---[0.172389ms] com.example.MyClass:level1()
            `---[0.099156ms] com.example.MyClass:level2()
                `---[0.102814ms] com.example.MyClass:level3()
                    `---[200.456ms] com.example.MyClass:level4()
```

结果分析：

**调用链显示**: 你可以看到MyMethod方法依次调用了level1、level2、level3和level4，并且每个方法的执行时间都被详细记录下来。

**耗时分析**: level4方法的执行时间最长，主要是由于我们在该方法中添加了Thread.sleep(200)来模拟耗时操作。

### watch命令的使用

我创建的一个模拟的model，方便大家理解。

```Java
package com.example;

public class PersonInfoModel {
    private String name;
    private int age;

    // getters and setters
}

public class MyModel {
    private String id;
    private PersonInfoModel personInfoModel;

    // getters and setters
}

public class MyClass {

    public String processData(MyModel model) {
        System.out.println("Processing data...");
        return "Processed: " + model.getPersonInfoModel().getName();
    }

    public static void main(String[] args) {
        MyClass myClass = new MyClass();
        MyModel model = new MyModel();
        PersonInfoModel personInfo = new PersonInfoModel();
        personInfo.setName("Alice");
        personInfo.setAge(25);
        model.setPersonInfoModel(personInfo);
        model.setId("12345");
        myClass.processData(model);
    }
}
```

为了深入监控`personInfoModel`的属性，使用的`watch`命令：

```Bash
watch com.example.MyClass processData '{params[0].personInfoModel, returnObj}' -x 3
```

参数解释

- **`params[0].personInfoModel`**: 这里的`params[0]`表示第一个参数（即`MyModel`对象），`.personInfoModel`表示访问`MyModel`中的`personInfoModel`属性。通过这种方式，可以深入到嵌套的`PersonInfoModel`对象中，监控其内部属性（例如`name`和`age`）。
- **`returnObj`**: 继续监控方法的返回值。
- **`-x 3`**: 深度为3，确保能够解析到`PersonInfoModel`的内部属性。如果需要更多层级的解析，可以适当增加此值。

当运行`processData`方法，并执行上述`watch`命令后，Arthas将输出类似如下的信息：

```Bash
method-com.example.MyClass.processData
params[0].personInfoModel = PersonInfoModel(name=Alice, age=25)
returnObj = "Processed: Alice"
```