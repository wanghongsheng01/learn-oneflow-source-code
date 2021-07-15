# Thread 
## 生产者 Actor 与 消费者 Actor 之间消息传递，msg 的流动轨迹
![image](https://user-images.githubusercontent.com/31394900/125795762-ae38987f-8574-4687-987c-dacb22ea08be.png)

当 Actor a 给 Actor b 发消息时，会判断 Actor b 是否在当前线程内，<br>
如果是，则将 Actor a 发送的消息直接压入当前 Thread 的 Local Msg Queue 中；<br
否则 Actor a 将消息发送给当前机器的 Actor Msg Bus （每个 Machine 都有自己的一个 Actor Msg Bus），Actor Msg Bus 判断接收者的 Actor 的 Thread 是否在本机上，<br>
如果是，则 Actor Msg Bus 会在本机上找到该 Thread，将消息传给该 Thread 的 Msg channel。<br>
如果不是，Actor a 将 msg 发送给本机 Actor Msg Bus，Actor Msg Bus 再把 msg 发送给本机传输数据的 Common Net，由 Actor b 机器的 Common Net 接收 msg，
传递给接收机的 Actor Msg Bus，Actor Msg Bus 再找到接收 Actor 的 Thread，将消息压入该 Thread 的 Msg channel 消息队列中。<br>

## 术语：
*  Actor Msg Bus ：每个 Machine 都有自己的一个 Actor Msg Bus，当同一机器跨 Thread 或不同机器上传递 msg 时，需要 Actor Msg Bus 传递数据
*  Local Msg Queue ：本机上发送 Actor 所在的当前 Thread 中存放 msg 的队列容器
* Msg channel : 本机上跨线程的 Thread 或不同机器上的 Thread 中存放 msg 的队列容器
* Common Net : 跨机传输 msg 的 “班车”，由发送/接收 Actor 所在机器的 Actor Msg Bus 负责与 Common Net  对接 msg 数据

## 总结：
生产者 Actor 与 消费者 Actor 之间消息传递，msg 流动轨迹总共分为 3 种情形：
1. 发送 Actor 与 接收 Actor 同 Thread: 将 msg 直接压入当前 Thread 的 Local Msg Queue 消息队列中，即发送 Actor -> msg -> 压入本机 Local Msg Queue 消息队列中。<br>
2. 同机不同 Thread: 发送 Actor -> msg -> 本机 Actor Msg Bus -> 找到接收 Actor 的 Thread -> 压入 Msg channel 消息队列中。<br>
3. 不同机器不同 Thread: 发送 Actor -> msg -> 本机 Actor Msg Bus -> 本机 Common Net -> 接收机 Common Net -> 接收机 Actor Msg Bus -> 接收 Thread -> 压入 Msg channel 消息队列中<br>

## Actor msg 的消费
从当前 Thread 的 Local Msg Queue 消息队列中取 msg，如果 Local Msg Queue 队列为空，则从 Msg channel 中取。
