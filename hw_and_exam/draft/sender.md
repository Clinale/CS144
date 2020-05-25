# lab guide reading

职责: 从 byteStream 转换成 tcp segment (ip/udp datagram)

整体思考模型: 和 receiver 一样,窗口,往窗口里填字节流

干脆也以字节流的方式进行ack和重传吧? 而不是segment , 但是处理起来好像比较复杂

```
• Keep track of the receiver’s window (processing incoming acknos and window sizes)

• Fill the window when possible, by reading from the ByteStream, creating new TCP
segments (including SYN and FIN flags if needed), and sending them

• Keep track of which segments have been sent but not yet acknowledged by the receiver—
we call these “outstanding” segments

• Re-send outstanding segments if enough time passes since they were sent, and they
haven’t been acknowledged yet

```

```
?Why am I doing this? The basic principle is to send whatever the receiver will allow
us to send (filling the window), and keep retransmitting until the receiver acknowledges
each segment. This is called “automatic repeat request” (ARQ). The sender divides the
byte stream up into segments and sends them, as much as the receiver’s window allows.
Thanks to your work last week, we know that the remote TCP receiver can reconstruct
the byte stream as long as it receives each index-tagged byte at least once—no matter
the order. The sender’s job is to make sure the receiver gets each byte at least once.
```

```

It’s important to remember that the receiver can be any implementation of a valid TCP receiver—it
won’t necessarily be your own TCPReceiver. One of the valuable things about Internet standards is how
they establish a common language between endpoints that may otherwise act very differently.
```

```
looking through its collection of outstanding
TCPSegments and deciding if the oldest-sent segment has been outstanding for too long
without acknowledgment (that is, without all of its sequence numbers being acknowledged).
If so, it needs to be retransmitted (sent again).
```

```
_initial_retransmission_timeout

需要实现一个 retransimission timer 计时器

有个 tick method, 利用好

Every time a segment containing data (nonzero length in sequence space) is sent
(whether it’s the first time or a retransmission), if the timer is not running, start it
running so that it will expire after RTO milliseconds (for the current value of RTO).
5. When all outstanding data has been acknowledged, turn off the retransmission timer.

看 RFC

```
# test case reading

全定义为闭区间 , 初始化ackedWindow大小为0 , ackedleft=0 , ackedright=-1
初始化unAckLeft=0 , unAckRight=-1

                    unacked
    acked      acked        unacked
[                              ]

property 

synced , fined

nextSeqno ,  //下一个要被发出去的字节的seqno

segMap ,  

his_window_size //对方的window_size

ackedWindowLeft,
ackedWindowRight,

unAckWindowRight,
unAckWindowLeft = ackedWindowRight+1,



func fill_window()
if (!synced)
    segment.syn = true
    segment.seqno = nextSeqno
    nextSeqno++;
    segment_out.push(seg);
    synced=true;
if (!fined && stream_in.input_ended())
    segment.fin = true
    segment.seqno = nextSeqno
    nextSeqno++;
    segment_out.push(seg);
    fined=true;
    
if (stream_in.size>0 and his_window_size>0)
    minSize = min(in.size,his_window_size)
    segment.data = stream_in.pop_output(minSize)
    expectAckno = nextSeqno = nextSeqno+minSize-1+1
    segMap[expectAckno] = pair<segment,nowTime> //doubted , 用于超时重传
    
    segment_out.push(seg)


func ack_received(ackno,win_size)

if (ackno>=nextSeqno)
    foreach(segMap as seg)
        if seg.expectAckno <= ackno
            del(segMap[ackno])
    
    nextSeqno = min(ackno,nextSeqno) +1
    window_size = win_size
    return true;    
else
    return false

   
//窗口里未被ack的字节数 , fin也算一个seqno !!
func bytes_in_flight()
//闭区间
    return unAckWindowRight - unAckWindowLeft;

func tick(ms_since_last_tick)
    foreach (segmap as seg)
        sendTime = seg.second
        //超出时间窗口,触发重传
        if sendTime+ms_since_last_tick >=sendTime+TIMEOUT_DFLT
            segment_out.push(seg.first)