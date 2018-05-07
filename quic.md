# QUIC所踩过的坑

## 不靠谱的max data frame
  - 当发包的字节数到达服务端协商的上限时(flow control)，服务端可以发送一个max xxx data帧来允许客户端扩大窗口。但是QUIC规定，不允许对ack-only packet进行相应，这意味着如果max xxx data 帧丢失，并且后面所有包都是ack only packet。客户端只能知道丢失了一个包，但不能通过ack 告知server 端自己丢了包，那么客户端将会被阻塞住。
  
## 不靠谱的ack frame
  - 在很多实现上没有实现对ack的追踪(特别是ats)。如果出现大量的丢失ack情况，会导致client端的大量伪重传。QUIC自09开始，需要追踪对包含ack frame 的包的确认。并且建议随机的发送max data 或者ping来避免ack-only 包.11之后规定,ack应当包含已经确认过的旧的包，来避免ack丢失的情况的伪重传。
  - ack 的传送时机在11之后的版本开始确立，在此之前，ack的传送只有两个策略，延迟发送和每个包一个ack。延迟发送会导致恢复状态过慢，每个包一个ack很显然发挥不了ack range 的优势。如RFC5681，ack应该每隔2个满packet(对应TCP的2 MSS)发送一次或者延迟25ms，这使得ack不至于过分延迟。并且乱序包应该更快的响应来加速恢复速度。
  
## 拥塞窗口的崩塌
  - 当我们在一次网络波动时丢失了多个包，那么每个丢包都会把窗口减半，并且后续宝遵循快恢复算法。很明显这样使得窗口一下次降低到一个很小的值，并且恢复起来相当慢。自09以后引入recovery pos，使得避免因为一次网络波动而丢失多个包(此前ngtcp2 这里有个bug导致排查了很久)。
  - 当由于收到ack，而认定某个包丢失时。滑动窗口会减半(cwnd = bytes_in_flight / 2)，这时很明显bytes_in_flight > cwnd。这意味着发送者不允许发送任何数据，直到bytes_in_flight降到cwnd以下。这将会是一个相当长的时间(目前尚未解决)。

## 被限制的RTO重传
  - 在某些实现上RTO重传的2个packet，受到流控的控制。一旦RTO发生当前bytes_in_flight必然大于cwnd, 这样这两个packet将会被阻塞到bytes_in_flight < cwnd。事实上很早以前的QUIC版本就规定RTO发送的2 Packet不应该受到cwnd控制，这样发送出去的2packet，可以引出server端的ack以加速恢复速度。

## 不靠谱的PMTU
  - TCP有自己的一套PMTU探测算法（RFC xxxx 我忘了）。但是目前（基本上所有实现）的实现都没有实现该算法。这使得如果发送的一个包的字节数大于当前路劲的MTU,那么这个包将会被拆分成很多小的IP包，提高了某个QUIC包的丢失概率。

## TCP的专用加速器
  - 目前有不少网卡支持TSO和TRO，这使得TCP的分包更为简单，快捷。但是QUIC没有

## 运营商的制裁
  - 实测过程中，内网环境中QUIC和TCP 测量出的RTT趋于相同分布。但是公网环境中TCP 80 比udp 4433 在高峰时测量出的RTT 快了10多倍(tcp 30-40ms quic 300-500ms)。这表明udp在公网传输比tcp慢很多。
