7.8. Stateless Reset
===========
服务器可以选择发送stateless reset来解决链接的异常状态。客户端持续发送给服务器非法数据（unable to properly continue the connection）可能会导致服务器宕机。如果客户端能够处理，服务器希望用CONNECTION_CLOSE 帧来发送一个致命错误。

为了支持这一功能，服务器在传送参数期间（transport parameters）携带stateless_reset_token 值。 这个值将会被加密，只会被服务器和客户端知晓。

如果服务器收到一个无法处理的包，就使发送以下格式的包：
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|0|C|K|  00001  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                     [Connection ID (64)]                      +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                   Stateless Reset Token (128)                 +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Random Octets (*)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

这个包必须使用短格式头部（short header form），并且这个头部必须包含最小的可能的packet number（shortest possible packet number encoding）。这样最小化了服务器发送的最后一个数据包与此数据包之间的感知时差。开头的8个字节将会被翻译成packet number。即使服务器可以使用不同的短头类型（short header form），指示不同的pakcet number长度. 者任然使得辨认出stateless reset更加容易。

如果发生stateless reset，服务器会复制connection id字段。如果客户端没有携带connection id 或者配置忽略connection id， 服务器将会忽略这一字段。

服务器会将包含在传送参数期间（transport parameters）的stateless reset token的值拿出， 放在short header 和connetion id 之后。

在Stateless Reset Token之后，服务器会用包含随机数的字节填充消息。

这样做的目的是为了尽可能的使得stateless reset包与普通包一样（无法辨认出）。

stateless reset不适合作为错误的提示信号。一端（An endpoint）如果想要发送错误信号的，它必须使用CONNECTION_CLOSE帧（如果客户端能处理的话）。

7.8.1. Detecting a Stateless Reset
-------
当客户端收到无法解密的，并且用short header格式的消息时，客户端会去检查这个消息是不是一个潜在的state reset消息。然后，客户端将连接ID​​后面的16个八位字节与服务器在其传输参数中提供的stateless reset token进行 固定时间比较（constant-time comparsion ??）比较。如果值相等，链接必须被终止，否则客户端可以抛弃这个帧。

7.8.2. Calculating a Stateless Reset Token
--
stateless reset token 必须很难被猜到。服务器可以为每一个已创建的链接生成一个随机数。但是，如果服务器丢失了链接，这会在同个集群的不同机器的协调和存储上存在难题。Stateless reset专门用来处理状态丢失的情况，所以这种处理是次优的。


11.1. Edge Cases and Other Considerations
=========
当处理stream和connection级别的流控时我们需要考虑许多边缘问题。我们必须给予足够的时间使得双端都必须统一flow control状态。如果一端确认了flow control状态，这一端可能会发送超过另一端可以接受的数据，
链接将会被关闭。

反过来，如果发送者认为自己被阻塞了并且接收方希望接收更多数据时，发送者会死锁，因为发送者永远也等不到MAX_DATA帧和MAX_STREAM_DATA帧。

当接收方收到RST_STREAM帧时，接收方会关闭相应的stream对应的状态，并且忽略以后该stream上的所有的数据。这可能导致双方无法同步，这是因为RST_STREAM帧可能乱序，而在RST_STREAM帧之前的数据可能正在路上（there may be further bytes in flight.）。发送方将会对该链接级别上发送的数据进行计算，但是接收方没有收到这些数据，就不放对相应的数据进行计算。接收端必须想办法去知道连接上发送的数据并进行跳帧。

为了避免这种情况，发送方必须在该RST_STREAM上包含最后字节地偏移。当接收方接收到RST_STREAM帧时，接收者可以知晓在RST_STREAM帧之前发送的字节数，接受者必须对发送的字节进行统计并调整链接级别的flow controller。

11.1.1. Response to a RST_STREAM
---------
当RST_STREAM 突然终止一个流时，是否对已经已经接收的数据处理是应用程序自己的事，但通常情况下，在收到RST_STREAM后，端点会选择停止在自己的方向发送数据。如果发送方明确表明未来的帧不会被处理，发送方可以在同一时刻发送一个STOP_SENDING帧

11.1.2. Data Limit Increments
--------
该文档将解释怎么去建议MAX_DATA，MAX_STREAM_DATA字节数。考虑一些场景，我们不能频繁的发送只有很少改动frame，因为frame有自己的开销。同时，不频繁的limits变化很大的frame也必须避免，因为这回倒是阻塞。因此，我们必须在这两者之间做一次折中。

类似于tcp的实现，接受者使用自动tunning状态机来调整频率和发送数量，发送数可以基于往返时延和接收app消费的数据的速率进行评估。

11.2. Stream Limit Increment
===
像flow control一样，这篇文档介绍如何去实现通过MAX_STREAM_ID控制流数。MAX_STREAM_ID构成最小开销，所以可以控制MAX_STREAM_ID来控制对端的并行数。

实现者可以在对端初始流关闭的时候增加最大流ID。接受者也可以根据当前环境状态空时最大流id。

11.2.1. Blocking on Flow Control
-----
当接收方超过flow control的最大限制并且没有收到MAX_DATA和MAX_STREAM_DATA帧，接收方将会被阻塞并且接收方必须发送BLOKED帧或者STREAM_BLOCKED帧。这些帧将会对接收方的debug有帮助，除此之外，他们不必做任何事。
在发送MAX_DATA或MAX_STREAM_DATA之前，接收方不应该等待BLOCKED或STREAM_BLOCKED帧，因为这样做意味着整个往返时延内发送发无法发送任何数据。

拥塞控制的最好方式，尽可能的避免发送发停止工作。为了避免阻塞发送方和统计存在的数据丢失的可能性，接收方必须在期望发送发阻塞之前发送MAX_DATA或者MAX_STREAM_DATA帧至少2个往返延迟。当发送方到达限制时，发送方只会发送BLOCKED或者STREAM_BLOCKED一次。发送发不能够对同一个限制发送多个BLOCKED或者STREAM_BLOCKED帧，除非原始帧对视。新的LOCKED or STREAM_BLOCKED只能在数据limit更新以后发送。

11.3. Stream Final Offset
========
Final Offset用来统计整个流的传输字节数。如果一个流被reset，Final Offset将会被携带在RST_STREAM帧里。否则Final Offset将会被携带在STREAM帧并且携带FIN标记。

当一个流进入“half-closed (remote)”状态，这一端将会取得Final Offset。如果这一个帧丢失，该端将会在stream 帧中取得Final Offset并进入该状态。

任何一端不能够发送到达或者超过Final Offset的数据。

一旦Final Offset被缺人，Final Offset不能过被改变。如果任何一端试图使用RST_STREAM或者STRAM帧去改变Final Offset，必须将其视为FINAL_OFFSET_ERROR错误。接收方必须把超过Final Offset的数据视为FINAL_OFFSET_ERROR错误，即使流已经被关闭。生成这些错误不是强制性的，这是因为该端必须对关闭的流保持Final Offset状态，这将导致很大程度的状态提交（ a significant state commitment）。

