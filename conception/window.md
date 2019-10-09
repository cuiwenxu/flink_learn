# window

- watermark和window的关系：**watermark是数据流的特性**，watermark是从数据流中extract timestamp，然后根据ExecutionConfig类中autoWatermarkInterval来定期生成的，**因其来自于数据流，所以是数据流自身的特性**，而窗口则不是数据流本身的，打个比方，watermark是流水线的产品附带的属性的话，window就是负责操作流水线的机器的属性。下面是flink官方文档的解释

```
For example, with an event-time-based windowing strategy that creates non-overlapping (or tumbling) windows every 5 minutes and has an allowed lateness of 1 min, Flink will create a new window for the interval between 12:00 and 12:05 when the first element with a timestamp that falls into this interval arrives, and it will remove it when the watermark passes the 12:06 timestamp.

```

- 键控流和非键控流：

在定义窗口前，首先要设定流是否为键控流，使用keyBy()方法可以将无限流拆分为逻辑键控流，如果未调用keyBy()则为非键控流。

对于键控流，可以传入事件的任何属性用作键，**键可以让窗口化计算可以由多个任务并行执行**，因为每个键控流都可以独立于其余逻辑流进行处理，引用同一键的所有元素被发送到同一并行任务。

对于非键控流，您的原始流将不会拆分为多个逻辑流，并且所有加窗逻辑将由单个任务执行，即并行度为1。

- 窗口分配器

在指定是否对流进行键控之后，下一步是定义窗口分配器。窗口分配器定义了如何将元素分配给窗口。

WindowAssigner负责将每个传入元素分配给一个或多个窗口。 Flink带有针对最常见用例的预定义窗口分配器，即滚动窗口，滑动窗口，会话窗口和全局窗口。您还可以通过扩展WindowAssigner类来实现自定义窗口分配器。所有内置窗口分配器（全局窗口除外）均基于时间将元素分配给窗口，时间可以是处理时间，也可以是事件时间。

滚动窗口每个小窗口是不重叠的，滑动窗口是可以重叠的。

