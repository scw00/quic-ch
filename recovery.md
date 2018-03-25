QUIC senders use both ack information and timeouts to detect lost packets, and this section provides a description of these algorithms. Estimating the network round-trip time (RTT) is critical to these algorithms and is described first.

QUIC发送者使用ack信息和超时来检测丢失的packet，这章将讲述这种算法，探测rtt是这个算法的根本，所以我们先介绍如何探测rtt

RTT is calculated when an ACK frame arrives by computing the difference between the current time and the time the largest newly acked packet was sent. If no packets are newly acknowledged, RTT cannot be calculated. When RTT is calculated, the ack delay field from the ACK frame SHOULD be subtracted from the RTT as long as the result is larger than the Min RTT. If the result is smaller than the min_rtt, the RTT should be used, but the ack delay field should be ignored.

当收到ack帧时，我们可以通过计算当前时间和最后的被确认的packet发送时间的差值来计算出当前rtt。一旦测量出RTT，只要rtt减去 ack delay的值大于最小的RTT，那么我们就用这个值更新RTT。如果这个值小于最小的rtt，那么该测量出的RTT可以是被使用，但是ack delay必须被忽略。

Min RTT is the minimum RTT measured over the connection, prior to adjusting by ack delay. Ignoring ack delay for min RTT prevents intentional or unintentional underestimation of min RTT, which in turn prevents underestimating smoothed RTT.

最小RTT是通过链接测量出的RTT最小值，最小RTT是ack delay之前的值。忽略比min RTT小的ack delay是为了防止有意攻击，或者无意的过度测量，否则会导致对smoothed rtt的过度测量。

Ack-based loss detection implements the spirit of TCP’s Fast Retransmit [RFC5681], Early Retransmit [RFC5827], FACK, and SACK loss recovery [RFC6675]. This section provides an overview of how these algorithms are implemented in QUIC.

基于ack的丢失探测实现了基本的TCP Fast Retransmit[RFC5681], Early Retransmit [RFC5827], FACK和SACK loss recovery [RFC6675]. 这章将说明如何在quic实现这些算法。

An unacknowledged packet is marked as lost when an acknowledgment is received for a packet that was sent a threshold number of packets (kReorderingThreshold) after the unacknowledged packet. Receipt of the ack indicates that a later packet was received, while kReorderingThreshold provides some tolerance for reordering of packets in the network.

当收到一个确认某个包A的ack 并且这个包A在一个与包B相差一个或多个kReorderingThreshold时，则包B被标记为丢失。接收ack意味着最后的packet被收到。kReorderingThreshold对网络上的包排序提供了一定的容忍性。

kReorderingThreshold的建议值是3

We derive this default from recommendations for TCP loss recovery [RFC5681] [RFC6675]. It is possible for networks to exhibit higher degrees of reordering, causing a sender to detect spurious losses. Detecting spurious losses leads to unnecessary retransmissions and may result in degraded performance due to the actions of the congestion controller upon detecting loss. Implementers MAY use algorithms developed for TCP, such as TCP-NCR [RFC4653], to improve QUIC’s reordering resilience, though care should be taken to map TCP specifics to QUIC correctly. Similarly, using time-based loss detection to deal with reordering, such as in PR-TCP, should be more readily usable in QUIC. Making QUIC deal with such networks is important open research, and implementers are encouraged to explore this space.

这个建议值是TCP loss recovery [RFC5681] [RFC6675]的建议值。这可以让我们探测到网络上的重排，让发送者可以他测出伪丢失。伪丢失将导致不必要的重传，可能会因为流控而导致性能下降。实现者可能需要TCP中的一些算法，比如TCP-NCR，来提高QUIC重排的弹性，但是必须正确的将tcp的特性映射到quic上。同样，基于时间的丢失探测可以被用作排序，如PR-TCP，这也同样对QUIC有效。是QUIC更好处理的这些场景是一项重要的研究，并且我们鼓励开发者去探索和发现这一领域。


3.2.2. Early Retransmit

Unacknowledged packets close to the tail may have fewer than kReorderingThreshold retransmittable packets sent after them. Loss of such packets cannot be detected via Fast Retransmit. To enable ack-based loss detection of such packets, receipt of an acknowledgment for the last outstanding retransmittable packet triggers the Early Retransmit process, as follows.

如果一个包未被确认且靠近尾部，那么可能只有少于kReorderingThreshold 的可被重传包在这个包之后发送。这样的丢失包无法送过Fast  Retransmit探测到。接收方接收到最后发送的可被重传的包的ack时，将会触发 Early Retransmit，这将启动基于ack的丢包探测。如下：

If there are unacknowledged retransmittable packets still pending, they should be marked as lost. To compensate for the reduced reordering resilience, the sender SHOULD set an alarm for a small period of time. If the unacknowledged retransmittable packets are not acknowledged during this time, then these packets MUST be marked as lost.

如果这个未被确认的可重传的包任然被阻塞，那么我们需要它标记为丢失。但是为了补偿我们降低了重排弹性(ps: 没有对收到的包进行重排，等候另外的可能乱续包)，发送者需要设置一个超时间隔较小的定时器。如果在此期间内还没有ack确认这个包，那么这个包必须标记为丢失。

An endpoint SHOULD set the alarm such that a packet is marked as lost no earlier than 1.25 * max(SRTT, latest_RTT) since when it was sent.

一端需要对这种被标记为丢失的包设置一个从这个包发送开始计时，不早于1.25 * max(SRTT, latest_RTT)的定时器。

Using max(SRTT, latest_RTT) protects from the two following cases:

使用max(SRTT, latest_RTT)有以下两个原因：

the latest RTT sample is lower than the SRTT, perhaps due to reordering where packet whose ack triggered the Early Retransit process encountered a shorter path;

1、lastest RTT样本小于SRTT，可能是由于某个包触发的ack触发的Early Retransit逻辑，并且这个重传包遇到一个更短的路径。

the latest RTT sample is higher than the SRTT, perhaps due to a sustained increase in the actual RTT, but the smoothed SRTT has not yet caught up.

2、lastest RTT样本大于SRTT，可能是由于真实RTT持续的增加，但是SRTT并没有捕获到。

The 1.25 multiplier increases reordering resilience. Implementers MAY experiment with using other multipliers, bearing in mind that a lower multiplier reduces reordering resilience and increases spurious retransmissions, and a higher multipler increases loss recovery delay.

可以用1.25因子来提升重排的弹性，实现者可能需要使用实验别的因子，但是要记住，小的因子会降低重排的弹性，增加因子可能会导致伪重传，更高的因子会提高的丢失探测的延迟。

This mechanism is based on Early Retransmit for TCP [RFC5827]. However, [RFC5827] does not include the alarm described above. Early Retransmit is prone to spurious retransmissions due to its reduced reordering resilence without the alarm. This observation led Linux TCP implementers to implement an alarm for TCP as well, and this document incorporates this advancement.

状态机源于TCP 的Early Retransmit[RFC5827]，然而[RFC5827]并没有上述对定时器的表述。tcp Early Retransmit由于降低重排的弹性和取消了定时器，很容易导致伪重传。这个现象使得linux的开发人员同样实现了一个定时器。


3.3. Timer-based Detection

Timer-based loss detection implements a handshake retransmission timer that is optimized for QUIC as well as the spirit of TCP’s Tail Loss Probe and Retransmission Timeout mechanisms.

基于定时器的丢失探测实现了握手重传定时器，它实现了基本的TCP的Tail Loss Probe 和Retransmission Timeout状态机。

3.3.1. Handshake Timeout
Handshake packets, which contain STREAM frames for stream 0, are critical to QUIC transport and crypto negotiation, so a separate alarm is used for them.

包含stream 0 的stream 帧的握手包对QUIC的传输和加密协商而言很重要，所以一个独立的定时器专门服务他们。

The initial handshake timeout SHOULD be set to twice the initial RTT.

initial 握手超时需要设置成初始RTT的两倍。

At the beginning, there are no prior RTT samples within a connection. Resumed connections over the same network SHOULD use the previous connection’s final smoothed RTT value as the resumed connection’s initial RTT.

一开始并没有RTT样本。复用链接应该使用同一网络下的前一个链接的最后的SRTT的值

If no previous RTT is available, or if the network changes, the initial RTT SHOULD be set to 100ms.

如果不是复用链接，或者网络发生改变，RTT初始值必须被设置成100ms。

When a handshake packet is sent, the sender SHOULD set an alarm for the handshake timeout period.

一旦发送者发送握手包，发送者需要设置握手的超时定时器。

When the alarm fires, the sender MUST retransmit all unacknowledged handshake data, by calling RetransmitAllUnackedHandshakeData(). On each consecutive firing of the handshake alarm, the sender SHOULD double the handshake timeout and set an alarm for this period.

当定时器超时，发送者需要重传所有未被确认的我收消息(通过调用RetransmitAllUnackedHandshakeData)。当连续的超时时，需要将超时设置为原来的2倍。

When an acknowledgement is received for a handshake packet, the new RTT is computed and the alarm SHOULD be set for twice the newly computed smoothed RTT.

当收到握手包的ack时，新的定时器需要设置成新算出的SRTT的两倍。

Handshake data may be cancelled by handshake state transitions. In particular, all non-protected data SHOULD no longer be transmitted once packet protection is available.

握手数据可能会被握手状态的轮转而取消。特别是，一旦保护包可用，非保护的包不能被发送

(TODO: Work this section some more. Add text on client vs. server, and on stateless retry.)

3.3.2. Tail Loss Probe
The algorithm described in this section is an adaptation of the Tail Loss Probe algorithm proposed for TCP [TLP].
这章说明TCP中的Tail Loss Probe 算法

A packet sent at the tail is particularly vulnerable to slow loss detection, since acks of subsequent packets are needed to trigger ack-based detection. To ameliorate this weakness of tail packets, the sender schedules an alarm when the last retransmittable packet before quiescence is transmitted. When this alarm fires, a Tail Loss Probe (TLP) packet is sent to evoke an acknowledgement from the receiver.

尾部发送的包很难被探测到，因为没有后续的包来触发基于ack的探测。为了解决这个问题，发送者为这个尾包设置一个定时器。当定时器触发时，尾包将被再次发送并等待接收者的确认。

The alarm duration, or Probe Timeout (PTO), is set based on the following conditions:

超时间隔PTO的设置基于以下两个条件：

PTO SHOULD be scheduled for max(1.5*SRTT+MaxAckDelay, kMinTLPTimeout)

1、PTO必须是max(1.5*SRTT+MaxAckDelay, kMinTLPTimeout)，如果RTO(Section 3.3.3)

If RTO (Section 3.3.3) is earlier, schedule a TLP alarm in its place. That is, PTO SHOULD be scheduled for min(RTO, PTO).
MaxAckDelay is the maximum ack delay supplied in an incoming ACK frame. MaxAckDelay excludes ack delays that aren’t included in an RTT sample because they’re too large and excludes those which reference an ack-only packet.

2、如果RTO太小，设置TLP超时定时器。既PTO = min(RTO, PTO)。MaxAckDelay是接收到的ack 帧中的最大的ack delay。MaxAckDelay用来排除大于RTT样本和指向只有ack frame的包的ack dealy。

QUIC diverges from TCP by calculating MaxAckDelay dynamically, instead of assuming a constant delayed ack timeout for all connections. QUIC includes this in all probe timeouts, because it assume the ack delay may come into play, regardless of the number of packets outstanding. TCP’s TLP assumes if at least 2 packets are outstanding, acks will not be delayed.

QUIC和TCP一样动态的计算MaxAckDelay，而不是对所有链接设置一个同一个值。QUIC将这个值运用到所有的超时探测中，因为他无论多少包在链路上，他都假设ack delay可能会改变。
TCP的TLP假设如果只是2个包在链路上，那么ack不会被延迟。

A PTO value of at least 1.5*SRTT ensures that the ACK is overdue. The 1.5 is based on [TLP], but implementations MAY experiment with other constants.

PTO的值至少为1.5*SRTT，这样保证了ACK在网络上过期，1.5是基于TLP协议，但是实现这可以测试别的值。

To reduce latency, it is RECOMMENDED that the sender set and allow the TLP alarm to fire twice before setting an RTO alarm. In other words, when the TLP alarm fires the first time, a TLP packet is sent, and it is RECOMMENDED that the TLP alarm be scheduled for a second time. When the TLP alarm fires the second time, a second TLP packet is sent, and an RTO alarm SHOULD be scheduled Section 3.3.3.

为了减少潜在因素，建议发送者在设置RTO超时之间设置TLP超时2次。换句话说，当TLP超时第一次触发时，发送TLP包，建议再次设置TLP超时。当第二次TLP超时触发，第二个TLP包发送。并且必须设置RTO超时。

A TLP packet SHOULD carry new data when possible. If new data is unavailable or new data cannot be sent due to flow control, a TLP packet MAY retransmit unacknowledged data to potentially reduce recovery time. Since a TLP alarm is used to send a probe into the network prior to establishing any packet loss, prior unacknowledged packets SHOULD NOT be marked as lost when a TLP alarm fires.

如果可能TLP包应该携带新的信息。如果新数据不可用或者新数据无法发送(flow control)，TLS数据可能会重传未确认的数据来减少恢复时间。因为TLP定时器被用于探测到包丢失之前，当TLP超市定时器触发，未被确认的包不应该被标记为丢失。

A sender may not know that a packet being sent is a tail packet. Consequently, a sender may have to arm or adjust the TLP alarm on every sent retransmittable packet.

发送者可能不知道这个包是一个尾包。因此，发送者可能需要对每一个可重传的包准备和调整TLP定时器。


3.3.3. Retransmission Timeout
A Retransmission Timeout (RTO) alarm is the final backstop for loss detection. The algorithm used in QUIC is based on the RTO algorithm for TCP [RFC5681] and is additionally resilient to spurious RTO events [RFC5682].

重传定时器用来辅助丢失探测，QUIC使用基于TCP[RFC5681]的算法，并且对伪RTO提供弹性。

When the last TLP packet is sent, an alarm is scheduled for the RTO period. When this alarm fires, the sender sends two packets, to evoke acknowledgements from the receiver, and restarts the RTO alarm.

当最后的TLP包发送时，设置RTO定时器。当RTO定时器触发，发送者发送两个包，来触发接收者的ack，并重启RTO定时器。

Similar to TCP [RFC6298], the RTO period is set based on the following conditions:

和TCP相同RTO超时事件基于以下两个条件：

When the final TLP packet is sent, the RTO period is set to max(SRTT + 4*RTTVAR + MaxAckDelay, kMinRTOTimeout)
When an RTO alarm fires, the RTO period is doubled.

1、当最后的TLP包发送时，RTO = max(SRTT + 4*RTTVAR + MaxAckDelay, kMinRTOTimeout)，当RTO定时器触发，RTO超时时间设置成原来的两倍。

The sender typically has incurred a high latency penalty by the time an RTO alarm fires, and this penalty increases exponentially in subsequent consecutive RTO events. Sending a single packet on an RTO event therefore makes the connection very sensitive to single packet loss. Sending two packets instead of one significantly increases resilience to packet drop in both directions, thus reducing the probability of consecutive RTO events.

2、发送者可能会遭遇到RTO超时带来的惩罚，并且这种惩罚以指数级增加(RTO多次超时)。所以发送一个包是的链接过于敏感，发送两个包以显著的提升链接的弹性，可以减少持续RTO事件。

QUIC’s RTO algorithm differs from TCP in that the firing of an RTO alarm is not considered a strong enough signal of packet loss, so does not result in an immediate change to congestion window or recovery state. An RTO alarm fires only when there’s a prolonged period of network silence, which could be caused by a change in the underlying network RTT.

QUIC RTO协议和TCP的区别：RTO超时并不能直接表明pakcet丢失，所以不能直接修改流控窗口和恢复状态。RTO超时只能说明这个包在网络中持续了很久，这可能导致网络RTT的改变。

QUIC also diverges from TCP by including MaxAckDelay in the RTO period. QUIC is able to explicitly model delay at the receiver via the ack delay field in the ACK frame. Since QUIC corrects for this delay in its SRTT and RTTVAR computations, it is necessary to add this delay explicitly in the TLP and RTO computation.

QUIC和TCP还有一个不同之处在于QUIC引入了MaxAckDelay。QUIC接收者可以通过ack 帧中ack delay 项明显的感受到延迟。因为QUIC他通过SRTT和RTTVAR的计算来修正这个delay(修正ack delay项？？？)，在TLP和RTO中添加这个delay很重要。

When an acknowledgment is received for a packet sent on an RTO event, any unacknowledged packets with lower packet numbers than those acknowledged MUST be marked as lost.

当字啊一个RTO超时期内收到这个ack时，任何比这个确认的包小的未被确认的包必须标记为lost。

A packet sent when an RTO alarm fires MAY carry new data if available or unacknowledged data to potentially reduce recovery time. Since this packet is sent as a probe into the network prior to establishing any packet loss, prior unacknowledged packets SHOULD NOT be marked as lost.

在由于一个RTO超时间而发送的包可能携带一些新的数据(可能的话)来减少恢复时间。因为这个包将作为一个在决定别的包丢失之前的的网络探测，之前的未被确认的包不能够被表姐为丢失。

A packet sent on an RTO alarm MUST NOT be blocked by the sender’s congestion controller. A sender MUST however count these bytes as additional bytes in flight, since this packet adds network load without establishing packet loss.

RTO定时器触发的packet不能被发送者的Congestion Controller阻塞。然而发送者需要将这些发送的字节加到在链路上的字节数，因为这些包在没有检测到packet丢失的情况下加入了网路。

3.4. Generating Acknowledgements
QUIC SHOULD delay sending acknowledgements in response to packets, but MUST NOT excessively delay acknowledgements of packets containing non-ack frames. Specifically, implementaions MUST attempt to enforce a maximum ack delay to avoid causing the peer spurious timeouts. The default maximum ack delay in QUIC is 25ms.

QUIC 应该延迟回应包中的确认消息，但是不能够过分的延迟不包含ack 帧的包的确认。特别是，实现者必须尝试强制ack delay的最大值，来避免对端的伪超时。默认最大ack延迟为25ms

An acknowledgement MAY be sent for every second full-sized packet, as TCP does [RFC5681], or may be sent less frequently, as long as the delay does not exceed the maximum ack delay. QUIC recovery algorithms do not assume the peer generates an acknowledgement immediately when receiving a second full-sized packet.


Out-of-order packets SHOULD be acknowledged more quickly, in order to accelerate loss recovery. The receiver SHOULD send an immediate ACK when it receives a new packet which is not one greater than the largest received packet number.

As an optimization, a receiver MAY process multiple packets before sending any ACK frames in response. In this case they can determine whether an immediate or delayed acknowledgement should be generated after processing incoming packets.
