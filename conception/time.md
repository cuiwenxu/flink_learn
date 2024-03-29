# flink的三种时间以及生成watermark的两种方式

- **event time**:事件时间是每个事件在其生产设备上发生的时间。该时间通常在它们进入Flink之前嵌入到记录中，并且可以从每个记录中提取事件时间戳。在事件时间中，时间的进度取决于数据，而不取决于任何挂钟（系统时间）。事件时间程序必须指定如何生成事件时间水印，这是信号事件时间进展的机制。

  在理想情况下，不管事件何时到达或它们的顺序如何，事件时间处理都将产生完全一致且确定的结果。但是，除非已知事件是按时间戳（按时间戳）到达的，否则事件时间处理会在等待无序事件时产生一定的延迟。由于只能等待有限的时间，因此这限制了确定性事件时间应用程序的可用性。

  假设所有数据都已到达，事件时间操作将按预期方式运行，即使在处理无序或迟到的事件或重新处理历史数据时，也会产生正确且一致的结果。例如，每小时事件时间窗口将包含所有带有落入该小时事件时间戳的记录，无论它们到达的顺序或处理的时间。

  请注意，有时当事件时间程序实时处理实时数据时，它们将使用一些处理时间操作，以确保它们及时进行。

- **process_time**:处理时间是指正在执行相应operator的机器的系统时间。

  当流式程序按处理时间运行时，所有基于时间的操作（如时间窗口）都将使用运行相应operator的计算机的系统时钟。处理时间是最简单的时间概念，不需要流和机器之间的协调。它提供了最佳的性能和最低的延迟。但是，在分布式和异步环境中，处理时间不能提供确定性，因为它容易受到记录到达系统（例如从消息队列）到达系统的速度，记录在系统内部操作员之间流动的速度的影响，以及中断（或其他方式）。

- **Ingestion time**:摄取时间是事件进入Flink的时间。 在源操作员处，每条记录都将源的当前时间作为时间戳记，并且后续基于时间的操作都将（例如时间窗口）引用该时间戳记。

  摄取时间从概念上讲介于事件时间和处理时间之间。 与处理时间相比，它的使用成本更高一些，但结果却更可预测。 由于摄取时间使用稳定的时间戳（在源处分配了一次），因此对记录的不同窗口操作将引用相同的时间戳，而在处理时间中，每个窗口操作员都可以将记录分配给不同的窗口（基于本地系统时钟和任何运输延误）。
**三种时间的概念如图所示**：
![flink time](time_clock.png)

## watermark生成的两种方式

- 定期生成watermark的方式:

  想要定期生成watermark,需要给让自定义类实现AssignerWithPeriodicWatermarks接口，重写extractTimestamp（提取时间戳）和getCurrentWatermark（生成watermark）这两个方法。分配时间戳并定期生成水印。通过ExecutionConfig.setAutoWatermarkInterval（...）定义生成水印的时间间隔（每n毫秒）。 每次都会调用分配者的getCurrentWatermark（）方法，如果返回的水印非空且大于前一个水印，则将发出新的水印。
```java
  public class MyTimestampAndWatermarkGenerator implements AssignerWithPeriodicWatermarks<String> {
  
          private long maxOutOfOrderness = 3500; // 3.5 seconds
          private long currentMaxTimestamp;
  
          @Nullable
          @Override
          public Watermark getCurrentWatermark() {
              return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
          }
  
          @Override
          public long extractTimestamp(String element, long previousElementTimestamp) {
              JSONObject jsonObject = JSON.parseObject(element);
              long create_time = Long.parseLong(jsonObject.get("create_time").toString());
              if (create_time > currentMaxTimestamp) {
                  currentMaxTimestamp = create_time;
              }
              return create_time;
          }
      }
```
- 根据数据源的特殊标记生成watermark
要根据数据源的特殊标记生成watermark生成新的水印时生成水印，需使用AssignerWithPunctuatedWatermarks。 对于此类，Flink将首先调用extractTimestamp（...）方法为该元素分配时间戳，然后立即在该元素上调用checkAndGetNextWatermark（...）方法。
```
public class PunctuatedAssigner implements AssignerWithPunctuatedWatermarks<MyEvent> {

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark checkAndGetNextWatermark(MyEvent lastElement, long extractedTimestamp) {
		return lastElement.hasWatermarkMarker() ? new Watermark(extractedTimestamp) : null;
	}
}
```

