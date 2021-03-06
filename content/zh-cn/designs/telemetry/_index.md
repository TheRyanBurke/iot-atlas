---
title: "遥测"
weight: 80
draft: true
personae: [foo, bar]
type: 'design'
layout: 'single'
---

<!-- {{< synopsis-telemetry >}} -->
从远程环境中的传感器收集数据，并将感测的测量值供IoT解决方案的其他组件使用。
<!--more-->

## 挑战
IoT解决方案需要可靠和安全地接收不同设备在远程环境中测量的数据，并可能使用到不同的协议。另外，一旦接收到测量数据，解决方案就需要处理和路由这些数据以供解决方案的组件来使用。

## 解决方案
IoT解决方案使用"遥测"设计，通过支持成熟的通信协议来确保在间歇性中断的网络环境下能够传送测量数据，同时提供在不同测量频率和数据量的情况下可扩展的数据接收能力，以及将数据路由给其他组件使用。

以下架构图展示的"遥测"设计可以提供这样的功能。

![Telemetry Design](telemetry.png)
([PPTx](/designs/iot-atlas-patterns.pptx))

### 步骤说明

1. 设备从在远离IoT解决方案环境中运行的传感器中获取测量值。
2. 设备向包含测量值的主题`telemetry/deviceID`发布消息。该消息通过传输协议发送到服务器提供的协议端点。
3. 服务器可以将一个或多个[规则]({{<ref"/glossary/vocabulary#rule">}})应用于消息，以便对部分或全部消息中的测量数据执行细粒度的路由。路由可以将消息发送到解决方案的另一个组件。

## 考虑点
当项目需要“来自设备的流式数据”时，通常会使用**遥测**设计。此外，在实施IoT解决方案时，*遥测*一词通常用作描述上述设计图的一种方式，同时用作简要描述从远程环境感测并传送数据到一个更大的解决方案时所内在的全部挑战。这些考虑集中在与实现上述设计图时相关联的主题上。

当实现上述设计时，请考虑如下问题：

#### 在IoT解决方案中，遥测消息所需的 *感测到洞察*(sense-to-insight) 或 *感测到行动*(sense-to-action) 处理延时是多少？

对于IoT解决方案中需要**微秒**或**毫秒**级别处理延时的，相应的处理应该在设备本身或是连接到设备的设备[网关]({{<ref "/designs/gateway" >}})上进行。

对于IoT解决方案中需要**秒**,**分钟**甚至是**小时**级别处理延时的，相应的处理默认应该在云端进行。通常来说，对于需要“秒”到“分钟”级别处理的消息，应该由直接连接到协议端点的组件来进行处理。在消息到达并满足一定的条件时，相应的组件会被触发并对消息进行处理。

对于从“分钟”到“小时”延时级别的遥测数据，应该以异步的方式进行处理。当消息到达并满足期望的条件时，相应的事件会被放在一个处理队列中并且有对应的组件进行必要的处理。一旦处理完成，该组件通常会发送一条消息到"处理完成"[消息主题]({{< ref "/glossary/vocabulary#message-topic" >}}).

#### 是否有经验教训可以使IoT解决方案中的遥测数据更易于处理？

**解决方案唯一的设备ID** - 每个设备在解决方案中都应该有*解决方案唯一*的ID。虽然这个ID并不一定真的要全球唯一，但每个设备需要在IoT解决方案中有永远唯一的ID。通过支持解决方案唯一的设备ID，IoT解决方案可以更好的处理和路由感测的数据，供解决方案内的组件所使用。

**尽早打时间戳** - 越早让感测数据获得在IoT解决方案中离散的时间戳，就越容易对感测数据进行细致入微的处理和分析

**闭合窗口计算** - 追踪设备的 `last_reported`(最后报告)时间戳可以确定是否/何时聚合窗口可以被视为*闭合*。然后在IoT解决方案中可以轻松且可信的缓存任何闭合窗口计算结果。这些缓存的计算结果经常可以极大的提高IoT解决方案中*感测到洞察(sense-to-insight)* 或 *感测到行动(sense-to-action)* 的处理性能。

#### 大消息应该如何处理
在这个设计中，大消息被定义为任何大于传输协议原生能支持的消息。对于大消息，需要回答一个额外的问题：是否可以使用辅助协议？

如果**是**，则推荐使用HTTP协议

如果**不是**,则大消息需要被切片，每个分片需要有一个独立的切片ID，同时每个切片都需要足够小，以便以能够使用传输协议进行发送

#### 当使用辅助协议时，应该如何发送大消息
如果大消息必须**尽可能快的传递**，则可以直接将大消息上传至一个全球可用，且具有强数据持久性的对象存储服务

如果大消息**可以被批量发送**, 那每个消息应该做为批量消息的一部分保存下来，直到批量消息可以被发送出去。由于设备上的存储资源通常是非常有限，对消息的批量处理应该考虑与设备做为[网关]({{<ref"/designs/gateway" >}})时同样的[算法权衡]({{< ref "/designs/gateway#how-should-data-be-processed-when-the-internet-is-unavailable" >}})

#### 什么是设备的采样和报告频率？

**采样频率** 是从连接的[传感器]({{< ref "/glossary/vocabulary#sensor" >}})上获取或采样感测数据的频率。

**报告频率** 是存储在设备上的采样数据发送到IoT解决方案的频率
  
设备上的代码会获取感测数据并将其排队以进行传递，或立即传递感测数据。这两种不同的行为通常被讨论为*采样频率和报告频率之间的差异*。当采样和报告频率相等且对齐时，所有感测数据将被立即传递。当两个频率不同时，必须考虑为入队数据选择正确的日志算法。

这两个频率的预期数值对于确定IoT解决方案的规模和成本是至关重要的。

#### 是否需要维持入站消息的顺序
首先，解决方案只有在绝对必要时，才应该依赖顺序  

如果**不要求**顺序，那解决方案可以在消息到达后立即进行处理

如果**要求**顺序，那么需要回答以下问题："在多长一段时间范围内，解决方案的组件需要有序的消息？"

如果答案是"在一个主题上小于1秒钟"，那解决方案可以从主题`foo`上将消息收集到一个缓冲区，然后每1秒钟缓冲区的消息被排序并且发送到另一个主题 `foo/ordered`。

如果上述答案是“大于1秒”，那IoT解决方案应该将每个记录都写入一个[有序存储]({{< ref "/glossary/vocabulary#ordered-store" >}})中。解决方案中任何**要求**消息总是有序的组件，可以从有序存储中读取并获得更新。

#### IoT解决方案中遥测的成本驱动因素有哪些？

通常，物联网解决方案中最常见的成本驱动因素是设备数量、设备采样和报告频率、必要的*感知到洞察力*或*感知到行动*的遥测处理延迟、[设备数据密度] ({{<ref"/glossary/vocabulary#device-data-density">}})以及[遥测存档]({{<ref "/designs/telemetry_archiving" >}})的保留时间。

#### 每个设备是否与其他设备在报告频率上“主动不对齐”？
当IoT解决方案中的所有设备都配置相同的报告频率时，则会出现一个具有重大影响的常见错误。为了避免隐藏在这个简单行为中的[相长干涉](http://en.wikipedia.org/w/index.php?title=Constructive_interference)，设备应该只在它唤醒并且在一段随机持续时间过后才开始进行报告。这种启动时间的随机性可以产生一个更为平滑的进入解决方案的感测数据流，防止因为设备在不可避免的区域网络中断或其他对解决方案有影响的中断后恢复通信时产生的相长干涉。

#### 当设备无法连接到其默认的IoT解决方案端点时应该做些什么？

**可预期的中断时长** - 当设备在可预期的时长内无法连接到默认的IoT解决方案端点，该设备应该按照配置进行*设备消息排队*。这与确定设备采样和报告频率的差异时的答案可能是一样的。此外，任何能够执行设备消息排队的设备都应该考虑与做为设备[网关]({{<ref"/designs/gateway">}})的设备相同的算法权衡。当本地存储不足以在预期持续时间内存储所有消息并且将影响所感测的数据时，则需要做出权衡。通常需要考虑的算法类别包括**[先进先出(FIFO)](https://zh.wikipedia.org/wiki/%E5%85%88%E9%80%B2%E5%85%88%E5%87%BA)**、**剔除(Culling)**和**聚合(Aggregate)**

**灾难级别的中断时长** - 当设备在灾难级别的时长内无法连接到默认的IoT解决方案端点，那就需要进行*区域切换(regional fail-over)*。为了实现这个方案，设备必须首先有一个预配置好的切换端点。接着设备需要重定向到切换端点，设备*已经注册*在新的区域上，并具有相应的凭证，设备可以像默认端点一样开始向新端点发送消息。否则，如果设备在新区域是*未注册*的话，那设备需要先在新区域完成[设备初始化]({{< ref "/designs/device_bootstrap" >}})，然后才能发送消息。

#### 如何在IoT解决方案中存储消息以便未来进行重放？

可以通过[遥测归档]({{<ref "/designs/telemetry_archiving">}})设计实现。

## 示例

### 遥测消息的生成、传送与路由

这是收集传感器数据并通过IoT解决方案进行发送所涉及的逻辑的详细示例。


#### 设备从传感器采样并生成消息

通过设备上的代码或是设备[网关]({{<ref "/designs/gateway">}})中运行的代码，设备以类似于以下伪代码的方式对传感器进行采样：

```python3
device_id = get_device_id()
while should_poll():  # loop until we should no longer poll sensors
    for sensor in list_of_sensors:
        # get timestamp of this 'sensor' reading
        ts = get_timestamp()
        # read the current sensor's value
        value = sensor.read_value()
        # add sensed data to message
        msq = create_message(device_id, sensor.get_id(), ts, value) 
        send_sensor_message(msg)  # send or enqueue message
    # sleep according to the sample frequency before next reading
    sleep(<duration>)
```

上面的`create_message`伪代码函数根据`device_id`、`sensor_id`
、时间戳`ts`和从传感器读取的`value`创建一条消息。


#### 设备对消息进行格式化
许多现有解决方案已经实现其消息格式。如果消息格式还在讨论中，则建议使用JSON。这是一个示例JSON消息格式：

``` json 
{
  "version": "2016-04-01",
  "deviceId": "<solution_unique_device_id>",
  "data": [ 
    {
      "sensorId": "<device_sensor_id>",
      "ts": "<utc_timestamp>",
      "value": "<actual_value>"
    }
  ]
}
```

#### 设备传递消息
一旦将感测到的数据放入消息中，设备就以报告频率将消息发布到远程协议端点。

当使用MQTT协议报告消息时，将使用主题发送消息。具有主题`telemetry/deviceID/example`的设备发送的消息将类似于以下伪代码。

```python
# get device ID of the device sending message
device_id = get_device_id()
# get the collection of unsent sensor messages
sensor_data = get_sensor_messages()
# the device's specific topic
topic = 'telemetry/' + device_id + '/example'
# loop through and publish all sensed data
while record in sensor_data:  
    mqtt_client.publish(topic, record, quality_of_service)
```

#### 消息发送至订阅者

每个发布的消息都会经过网络到达协议端点。消息被接收后，服务器会将每条消息提供给感兴趣的组件。组件会订阅特定的消息主题。

除了让IoT解决方案中的组件直接订阅消息主题外，一些IoT解决方案还有一个规则引擎，允许规则引擎订阅入站消息。然后，在逐个消息的基础上，规则引擎中的规则可以处理消息或将消息定向到IoT解决方案中的其他组件。
