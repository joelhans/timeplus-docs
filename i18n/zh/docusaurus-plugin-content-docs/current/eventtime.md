# _tp_time（事件时间）

## 所有流数据都应有事件时间

流是数据存在的地方，每个数据包含一个 `_tp_time` 列作为事件时间。 Timeplus 将此属性作为事件的一个重要特征。

事件时间用来确定事件发生的时间，例如一个人生日。  它可以是下单时的确切时间戳，用户登录系统时的确切时间戳，发生错误时的确切时间戳，或者 IoT 设备报告其状态时的确切时间戳。 如果事件中没有合适的时间戳属性，Timeplus 将根据数据摄取时间生成事件时间。

默认情况下， `_tp_time` 列在 `datetime64(3, 'UTC')` 类型以毫秒为精度。 您也可以在 `日期时间` 类型下创建它，并以秒为精度。

When you are about to create a new stream, please choose the right column as the event time. If no column is specified, then Timeplus will use the current timestamp as the value of `_tp_time` It's not recommended to rename a column as _tp_time at the query time, since it will lead to unexpected behaviour, specially for [Time Travel](usecases#s-time-travel).

## 为什么事件时间受到不同的处理

事件时间几乎在任何地方在 Timeplus 数据处理和分析工作流程中使用：

* 在执行基于时间窗口的聚合时， 例如 [tumble](functions_for_streaming#tumble) 或 [hop](functions_for_streaming#hop) 以获取每次窗口中的下载数据或外部数据， Timeplus 将使用事件时间来决定某些事件是否属于特定窗口。
* 在这种具有时间敏感性的分析中，事件时间也用来识别不合顺序的事件或较晚的事件， 并丢弃它们以便及时获得串流洞察力。
* 当一个数据流与另一个数据流合并时，事件时间是整理数据的钥匙，而不必期望两个事件在完全相同的毫秒内发生。
* 事件时间也发挥重要作用来设备数据在流中保存的时间。

## 如何指定事件时间

### 在数据摄取过程中指定

当您 [摄取数据](ingestion) 到 Timeplus 时，您可以在数据中指定一个属性来最能代表事件时间。 即使该属性是在 `字符串` 类型中，Timeplus 将自动转换为时间戳以便进一步处理。

如果您不在向导中选择属性，则 Timeplus 将使用摄取时间来显示事件时间。 例如：当 Timeplus 接收数据时。 这可能对大多数静态或维数据很有用，例如带有邮政编码的城市名称。

### 在查询时指定

[tumble](functions_for_streaming#tumble) 或 [hop](functions_for_streaming#hop) 窗口函数将可选参数作为事件时间列。 默认情况下，我们将使用每个数据中的事件时间。 然而，您也可以指定一个不同的列作为事件时间。

以出租车乘客为例。 数据流可以是

| 车号   | 用户ID | 行程开始                | 行程结束                | 费用 |
| ---- | ---- | ------------------- | ------------------- | -- |
| c001 | u001 | 2022-03-01 10:00:00 | 2022-03-01 10:30:00 | 45 |

数据可能来自 Kafka 主题。 配置完成后，我们可以将 `trip_end` 设置为（默认）事件时间。 所以，如果我们想要在每个小时内找出多少乘客，我们就可以这样运行查询

```sql
select count(*) from tumble(taxi_data,1h) group by window_end
```

此查询使用 `行程开始` ，默认事件时间来运行聚合。 如果旅客在午夜时00时01分结束行程，则行程将包括在00时00分-00时59时窗内。

在某些情况下，作为分析师，您可能想关注每小时有多少乘客上了出租车，而不是离开出租车，您可以设置 `行程结束` 作为查询的事件时间，通过 `tumblet(taxi_data,trip_end,1h)`

完整查询：

```sql
select count(*) from tumble(taxi_data,trip_end,1h) group by window_end
```

