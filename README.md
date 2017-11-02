
7.7.1. Privacy Implications of Connection Migration
---

在多个网络路径上使用稳定的Connection ID 可以使被动的观察这些路径之间的相互联系。client在不同网络路径间的移动往往不希望只希望和server之间保持互动（抛弃过去的路径）。server可以发送NEW_CONNECTION_ID消息来提供一个无关联的connection id。
这个id 可以被用作client希望显示的抛弃另一方的链接。


client 可能需要再收到server response 之前，在不同网络路径上发送多个包。为了保证client 不再与对方相关联。我们需要一个新的Connection id和packet number gap 对应对应每一个网络路径。为了支持这些，
server 发送多个NEW_CONNECTION_ID消息。每一个NEW_CONNECTION_ID消息用sequence 标记。 NEW_CONNECTION_ID中的connection id 必须按照顺序使用。


希望通过改变网络打破链接的客户端必须使用必须使用server提供的connection id。 同时，通过一个外部无法预测的计算（见7.7.1.1），自增每一个packet sequence。客户可能跳过connection ID，但是必须保证提交对应Connection id相关联的packet number 范围， 用来跳过不是该Connection id 的链接请求。

server 接收到被标记为新connection id 的packet可以通过增加累计的packet number gap来恢复预期的packet number。server必须抛弃包含比建议跟小的gao的packet。

例如，server 在新的Connection id上提供7个packet number 间隙。如果在前一个链接id上受到packet 10， 下一个预期packet必须为新连接的packet 18。新连接上packet 17 必须被抛弃。

7.7.1.1. Packet Number Gap
----
为了避免连锁，packet number 间隙必须是外部随机的外部无法确认的。一个connection id 的packe number 间隙的序号可以通过将sequence number编码成大段32位整型来计算：

```
Gap = HKDF-Expand-Label(packet_number_secret,
                        "QUIC packet sequence gap", sequence, 4)
```

HKDF-Expand-Label的输出被解释为大端数字。“packet_number_secret”由tls的key交换提供。如[QUIC-TLS] 5.6章的描述。

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

8.13. ACK Frame
=====

接收方发送ack帧来通知发送方这个包已经被接受和处理，或者表明哪些包丢失。ack帧包含1-256个blocks。ack blocks包含一组确认包。实现者不能够对只包含ack block的包发送ack包。虽然我们会再别的ack包中包含这个ack信息。

为了限制哪些没有被接收端接收到的ack blocks。接收端必须必须追踪每一个背对端确认的ack帧。同一个包不能够确认两遍。

如果接收方只发送ack帧，那么接收方永远也不会受到该ack帧相应的ack包。随机发送 MAX_DATA or MAX_STREAM_DATA 帧来标记数据已经被接受可以保证对端确认数据。否则一端可以每隔一个rtt发送一次 PING帧来获取确认帧。

为了限制接收端状态和ack 帧大小，接收端可以限制发送ack blocks数量。即使没有收到ack确认帧，接收端也可以收集ack blocks直到到达limit。但是这能会导致不必要的重传。

和tcp sacks不同的是，quic ack blocks是无法撤销的。一旦packet被确认，即使是在未来的某个ack中没有出现该packet的确认证，该确认依然有效。

客户端不能够确认Version Negotiation or Server Stateless Retry packets。这些帧的packet number是有client选择的，不是客户端逊则的。

QUIC ACK帧包含最多255个时间戳的时间戳部分。时间戳可以有利于更好的传输控制，但是不是现阶段loss recovery的必须品，并且求得时间戳价值有限，所以无法保证每一个时间戳都将被发送发收到。接收方应该为每个包和可重传帧的数据包发送一次时间戳。接收方可以为非重传包发送timestamps，但是不能为没有保护的包发送timestamps。

发送者可以有意地跳过分组号码，以将熵引入到连接中（introduce entropy into the connection），来避免随机ack攻击。当接收到未被发送的的packet的确认帧时，发送方应当关闭连接。ack帧格式很容易辨认出哪些包丢失，1到255之间的数据包编号有效地提供高达8位的有效熵。这对确认攻击起到保护作用。


The type byte for a ACK frame contains embedded flags, and is formatted as 101NLLMM. These bits are parsed as follows:
The first three bits must be set to 101 indicating that this is an ACK frame.
The N bit indicates whether the frame contains a Num Blocks field.
The two LL bits encode the length of the Largest Acknowledged field. The values 00, 01, 02, and 03 indicate lengths of 8, 16, 32, and 64 bits respectively.
The two MM bits encode the length of the ACK Block Length fields. The values 00, 01, 02, and 03 indicate lengths of 8, 16, 32, and 64 bits respectively.
An ACK frame is shown below.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|[Num Blocks(8)]|   NumTS (8)   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Largest Acknowledged (8/16/32/64)            ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        ACK Delay (16)         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     ACK Block Section (*)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Timestamp Section (*)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
Figure 7: ACK Frame Format
```
The fields in the ACK frame are as follows:
Num Blocks (opt):
An optional 8-bit unsigned value specifying the number of additional ACK blocks (besides the required First ACK Block) in this ACK frame. Only present if the ‘N’ flag bit is 1.
Num Timestamps:
An unsigned 8-bit number specifying the total number of <packet number, timestamp> pairs in the Timestamp Section.
Largest Acknowledged:
A variable-sized unsigned value representing the largest packet number the peer is acknowledging in this packet (typically the largest that the peer has seen thus far.)
ACK Delay:
The time from when the largest acknowledged packet, as indicated in the Largest Acknowledged field, was received by this peer to when this ACK was sent.
ACK Block Section:
Contains one or more blocks of packet numbers which have been successfully received, see Section 8.13.1.
Timestamp Section:
Contains zero or more timestamps reporting transit delay of received packets. See Section 8.13.2.
8.13.1. ACK Block Section
The ACK Block Section contains between one and 256 blocks of packet numbers which have been successfully received. If the Num Blocks field is absent, only the First ACK Block length is present in this section. Otherwise, the Num Blocks field indicates how many additional blocks follow the First ACK Block Length field.
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|              First ACK Block Length (8/16/32/64)            ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  [Gap 1 (8)]  |       [ACK Block 1 Length (8/16/32/64)]     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  [Gap 2 (8)]  |       [ACK Block 2 Length (8/16/32/64)]     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                             ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  [Gap N (8)]  |       [ACK Block N Length (8/16/32/64)]     ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
Figure 8: ACK Block Section
```
The fields in the ACK Block Section are:
First ACK Block Length:
An unsigned packet number delta that indicates the number of contiguous additional packets being acknowledged starting at the Largest Acknowledged.
Gap To Next Block (opt, repeated):
An unsigned number specifying the number of contiguous missing packets from the end of the previous ACK block to the start of the next. Repeated “Num Blocks” times.
ACK Block Length (opt, repeated):
An unsigned packet number delta that indicates the number of contiguous packets being acknowledged starting after the end of the previous gap. Repeated “Num Blocks” times.
8.13.2. Timestamp Section
The Timestamp Section contains between zero and 255 measurements of packet receive times relative to the beginning of the connection.
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
| [Delta LA (8)]|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    [First Timestamp (32)]                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|[Delta LA 1(8)]| [Time Since Previous 1 (16)]  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|[Delta LA 2(8)]| [Time Since Previous 2 (16)]  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|[Delta LA N(8)]| [Time Since Previous N (16)]  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
Figure 9: Timestamp Section

```
The fields in the Timestamp Section are:
Delta Largest Acknowledged (opt):
An optional 8-bit unsigned packet number delta specifying the delta between the largest acknowledged and the first packet whose timestamp is being reported. In other words, this first packet number may be computed as (Largest Acknowledged - Delta Largest Acknowledged.)
First Timestamp (opt):
An optional 32-bit unsigned value specifying the time delta in microseconds, from the beginning of the connection to the arrival of the packet indicated by Delta Largest Acknowledged.
Delta Largest Acked 1..N (opt, repeated):
This field has the same semantics and format as “Delta Largest Acknowledged”. Repeated “Num Timestamps - 1” times.
Time Since Previous Timestamp 1..N(opt, repeated):
An optional 16-bit unsigned value specifying time delta from the previous reported timestamp. It is encoded in the same format as the ACK Delay. Repeated “Num Timestamps - 1” times.
The timestamp section lists packet receipt timestamps ordered by timestamp.
8.13.2.1. Time Format
DISCUSS_AND_REPLACE: Perhaps make this format simpler.
The time format used in the ACK frame above is a 16-bit unsigned float with 11 explicit bits of mantissa and 5 bits of explicit exponent, specifying time in microseconds. The bit format is loosely modeled after IEEE 754. For example, 1 microsecond is represented as 0x1, which has an exponent of zero, presented in the 5 high order bits, and mantissa of 1, presented in the 11 low order bits. When the explicit exponent is greater than zero, an implicit high-order 12th bit of 1 is assumed in the mantissa. For example, a floating value of 0x800 has an explicit exponent of 1, as well as an explicit mantissa of 0, but then has an effective mantissa of 4096 (12th bit is assumed to be 1). Additionally, the actual exponent is one-less than the explicit exponent, and the value represents 4096 microseconds. Any values larger than the representable range are clamped to 0xFFFF.
8.13.3. ACK Frames and Packet Protection
ACK frames that acknowledge protected packets MUST be carried in a packet that has an equivalent or greater level of packet protection.
Packets that are protected with 1-RTT keys MUST be acknowledged in packets that are also protected with 1-RTT keys.
A packet that is not protected and claims to acknowledge a packet number that was sent with packet protection is not valid. An unprotected packet that carries acknowledgments for protected packets MUST be discarded in its entirety.
Packets that a client sends with 0-RTT packet protection MUST be acknowledged by the server in packets protected by 1-RTT keys. This can mean that the client is unable to use these acknowledgments if the server cryptographic handshake messages are delayed or lost. Note that the same limitation applies to other data sent by the server protected by the 1-RTT keys.
Unprotected packets, such as those that carry the initial cryptographic handshake messages, MAY be acknowledged in unprotected packets. Unprotected packets are vulnerable to falsification or modification. Unprotected packets can be acknowledged along with protected packets in a protected packet.
An endpoint SHOULD acknowledge packets containing cryptographic handshake messages in the next unprotected packet that it sends, unless it is able to acknowledge those packets in later packets protected by 1-RTT keys. At the completion of the cryptographic handshake, both peers send unprotected packets containing cryptographic handshake messages followed by packets protected by 1-RTT keys. An endpoint SHOULD acknowledge the unprotected packets that complete the cryptographic handshake in a protected packet, because its peer is guaranteed to have access to 1-RTT packet protection keys.
For instance, a server acknowledges a TLS ClientHello in the packet that carries the TLS ServerHello; similarly, a client can acknowledge a TLS HelloRetryRequest in the packet containing a second TLS ClientHello. The complete set of server handshake messages (TLS ServerHello through to Finished) might be acknowledged by a client in protected packets, because it is certain that the server is able to decipher the packet.

11. Flow Control
---------------

我们又必要限制在任何时间内发送者发送数据量，所以为了避免发送者发送大量的数据压倒慢的接收者，避免怀有恶意的发送者消耗大量接收者的资源。这一章节将介绍quic flow control状态机。

QUIC employs a credit-based flow-control scheme similar to HTTP/2’s flow control [RFC7540]. A receiver advertises the number of octets it is prepared to receive on a given stream and for the entire connection. This leads to two levels of flow control in QUIC: (i) Connection flow control, which prevents senders from exceeding a receiver’s buffer capacity for the connection, and (ii) Stream flow control, which prevents a single stream from consuming the entire receive buffer for a connection.
A data receiver sends MAX_STREAM_DATA or MAX_DATA frames to the sender to advertise additional credit. MAX_STREAM_DATA frames send the the maximum absolute byte offset of a stream, while MAX_DATA sends the maximum sum of the absolute byte offsets of all streams other than stream 0.
A receiver MAY advertise a larger offset at any point by sending MAX_DATA or MAX_STREAM_DATA frames. A receiver MUST NOT renege on an advertisement; that is, once a receiver advertises an offset, it MUST NOT subsequently advertise a smaller offset. A sender could receive MAX_DATA or MAX_STREAM_DATA frames out of order; a sender MUST therefore ignore any flow control offset that does not move the window forward.
A receiver MUST close the connection with a FLOW_CONTROL_ERROR error (Section 12) if the peer violates the advertised connection or stream data limits.
A sender MUST send BLOCKED frames to indicate it has data to write but is blocked by lack of connection or stream flow control credit. BLOCKED frames are expected to be sent infrequently in common cases, but they are considered useful for debugging and monitoring purposes.
A receiver advertises credit for a stream by sending a MAX_STREAM_DATA frame with the Stream ID set appropriately. A receiver could use the current offset of data consumed to determine the flow control offset to be advertised. A receiver MAY send MAX_STREAM_DATA frames in multiple packets in order to make sure that the sender receives an update before running out of flow control credit, even if one of the packets is lost.
Connection flow control is a limit to the total bytes of stream data sent in STREAM frames on all streams. A receiver advertises credit for a connection by sending a MAX_DATA frame. A receiver maintains a cumulative sum of bytes received on all streams, which are used to check for flow control violations. A receiver might use a sum of bytes consumed on all contributing streams to determine the maximum data limit to be advertised.


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
当发送方超过flow control的最大限制并且没有收到MAX_DATA和MAX_STREAM_DATA帧，发送方将会被阻塞并且发送方必须发送BLOKED帧或者STREAM_BLOCKED帧。这些帧将会对接收方的debug有帮助，除此之外，他们不必做任何事。
在发送MAX_DATA或MAX_STREAM_DATA之前，接收方不应该等待BLOCKED或STREAM_BLOCKED帧，因为这样做意味着整个往返时延内发送发无法发送任何数据。

拥塞控制的最好方式，尽可能的避免发送发停止工作。为了避免阻塞发送方和统计存在的数据丢失的可能性，接收方必须在期望发送发阻塞之前发送MAX_DATA或者MAX_STREAM_DATA帧至少2个往返延迟。当发送方到达限制时，发送方只会发送BLOCKED或者STREAM_BLOCKED一次。发送发不能够对同一个限制发送多个BLOCKED或者STREAM_BLOCKED帧，除非原始帧对视。新的LOCKED or STREAM_BLOCKED只能在数据limit更新以后发送。

11.3. Stream Final Offset
========
Final Offset用来统计整个流的传输字节数。如果一个流被reset，Final Offset将会被携带在RST_STREAM帧里。否则Final Offset将会被携带在STREAM帧并且携带FIN标记。

当一个流进入“half-closed (remote)”状态，这一端将会取得Final Offset。如果这一个帧丢失，该端将会在stream 帧中取得Final Offset并进入该状态。

任何一端不能够发送到达或者超过Final Offset的数据。

一旦Final Offset被缺人，Final Offset不能过被改变。如果任何一端试图使用RST_STREAM或者STRAM帧去改变Final Offset，必须将其视为FINAL_OFFSET_ERROR错误。接收方必须把超过Final Offset的数据视为FINAL_OFFSET_ERROR错误，即使流已经被关闭。生成这些错误不是强制性的，这是因为该端必须对关闭的流保持Final Offset状态，这将导致很大程度的状态提交（ a significant state commitment）。

