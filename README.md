## MQTT协议消息流

### 消息重发策略

在`QoS` > 0情况下，`PUBLISH`、`PUBREL`、`SUBSCRIBE`、`UNSUBSCRIBE`等类型消息在发送者发送完之后，需要等待一个响应消息，若在一个指定时间段内没有收到，发送者可能需要重试。重发的消息，要求`DUP`标记要设置为1.

等待响应的超时应该在消息成功发送之后开始算起，并且等待超时应该是可以配置选项，以便在下一次重试的时候，适当加大。比如第一次重试超时10秒，下一次可能为20秒，再一次重试可能为60秒呢。当然，还要有一个重试次数限制的。

还有一种情况，客户端重新连接，但未在可变头部中设置clean session标记，但双方(客户端和服务器端)都应该重试先前未发送的动态消息（in-flight messages）。客户端不被强制要求发送未被确认的消息，但服务器端就得需要重发那些未被去确认的消息。

- `QoS level`决定的消息流`QoS level`为`Quality of Service level`的缩写，翻译成中文，服务质量等级。

   MQTT 3.1协议在"4.1 `Quality of Service levels and flows`"章节中，仅仅讨论了客户端到服务器的发布流程，不太完整。因为决定消息到达率，能够提升发送质量的，应该是服务器发布PUBLISH消息到订阅者这一消息流方向。

- QoS level 0

```
- 至多发送一次，发送即丢弃。没有确认消息，也不知道对方是否收到。
- 针对的消息不重要，丢失也无所谓。
- 网络层面，传输压力小。
```
   ![](../image/QoS_level_0.png)

   ```sequence
   participant publisher
   participant broker
   participant subscriber
   publisher->>broker:PUBLISH[QoS=0]
   broker-->subscriber:PUBLISH
   publisher-->>publisher:Delete message
   ```

- QoS level 1
```
- 所有QoS level 1都要在可变头部中附加一个16位的消息ID。
- SUBSCRIBE和UNSUBSCRIBE消息使用QoS level 1。
- 针对消息的发布，Qos level 1，意味着消息至少被传输一次。
```

   发送者若在一段时间内接收不到PUBACK消息，发送者需要打开DUB标记为1，然后重新发送PUBLISH消息。因此会导致接收方可能会收到两次PUBLISH消息。针对客户端发布消息到服务器的消息流：

   ![](../image/QoS_level_1.png)

   针对服务器发布到订阅者的消息流：

   ![](../image/QoS_level_sub.png)

   ```mermaid
   sequenceDiagram
   participant publisher
   participant broker
   participant subscriber
   publisher->>publisher:Store message
   publisher->>broker:PUBLISH[QoS=1]
   broker->>broker:Store message
   broker->>subscriber:PUBLISH
   broker->>broker:Delete message
   broker->>publisher:PUBACK
   publisher-->publisher:Delete message
   ```

   发布者（客户端/服务器）若因种种异常接收不到PUBACK消息，会再次重新发送PUBLISH消息，同时设置DUP标记为1。接收者以服务器为例，这可能会导致服务器收到重复消息，按照流程，broker（服务器）发布消息到订阅者（会导致订阅者接收到重复消息），然后发送一条PUBACK确认消息到发布者。

   在业务层面，或许可以弥补MQTT协议的不足之处：重试的消息ID一定要一致接收方一定判断当前接收的消息ID是否已经接受过但一样不能够完全确保，消息一定到达了。

- QoS level 2

```
- 仅仅在PUBLISH类型消息中出现，要求在可变头部中要附加消息ID。
- 级别高，通信压力稍大些，但确保了仅仅传输接收一次。
```

   Publisher -> Server方向：

   ![](../image/QoS_level2_client.png)

   Server -> Subscriber:

   ![](../image/QoS_level2_server.png)

   ```mermaid
   sequenceDiagram
   participant publisher
   participant broker
   participant subscriber
   publisher->>publisher:Store message
   publisher->>broker:PUBLISH[QoS=1]
   broker->>broker:Store message
   broker->>subscriber:PUBLISH
   broker->>publisher:PUBREC
   publisher->>broker:PUBREL
   broker->>broker:Delete message
   broker->>publisher:PUBACK
   publisher-->publisher:Delete message
   ```

   Server端采取的方案a和b，都包含了何时消息有效，何时处理消息。两个方案二选一，Server端自己决定。但无论死采取哪一种方式，都是在QoS level 2协议范畴下，不受影响。若一方没有接收到对应的确认消息，会从最近一次需要确认的消息重试，以便整个（QoS level 2）流程打通。

### 消息顺序

消息顺序会受许多因素的影响，但对于服务器程序，必须保证消息传递流程的每个阶段要和开始的顺序一致。例如，在QoS level 2定义的消息流中，PUBREL流必须和PUBLISH流具有相同的顺序发送：

   ![](../image/msg_seq.png)

   流动消息(in-flight messages)数量允许有一个可保证的效果：

   - 在流动消息(in-flight)窗口1中，每个传递流在下一个流开始之前完成。这保证消息以提交的顺序传递
   - 在流动消息(in-flight)大于1的窗口，只能在QoS level内被保证消息的顺序

### 消息的持久化

在MQTT协议中，PUBLISH消息固定头部RETAIN标记，只有为1才要求服务器需要持久保存此消息，除非新的PUBLISH覆盖。

对于持久的、最新一条PUBLISH消息，服务器不但要发送给当前的订阅者，并且新的订阅者(new subscriber，同样需要订阅了此消息对应的Topic name）会马上得到推送。

Tip：新来乍到的订阅者，只会取出最新的一个RETAIN flag = 1的消息推送，不是所有。

### 消息流的编码/解码

MQTT协议中，由目前定义的14种类型消息在客户端和服务器端之间数据进行交互。若以JAVA语言构建MQTT服务器，可选择Netty作为基础。

在Netty中，数据的进入和流出，代表了一次完整的交互。无论是要进入的还是要流出的数据(单独以服务器为例)，都可看做字节流。若把每种类型消息抽象为一个具体对象，那么处理起来就不难了。

客户端->服务器，进入的字节流，逐个字节/单位读取，可还原成一个具体的消息对象（解码的过程）。

要发送到客户端的消息对象，转换（编码）成字节流，然后由TCP通道流转到接收者。
