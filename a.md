#### TCP的介绍

## python

```Python
# 时间复杂度：O(logn)
def search(array, val):
    length = len(array)
    left = 0
    right =  length - 1
    while left <= right:
        mid = (right + left) // 2 # 需整除2，获取下标
        value = array[mid]
        if value == val:
            return mid
        elif value < val: # 值在右边
            left = mid + 1
        else: # 值在左边
            right = mid - 1
    else:
        return None
li = list(range(1, 9))
print(search(li, 5))
# 结果
# [1, 2, 3, 4, 5, 6, 7, 8]
# 4

def select_sort(array):
    for i in range(len(array) - 1):
        min_loc = i  # 无序区最小值下标
        for j in range(i, len(array)):
            if array[j] < array[min_loc]:
                min_loc = j
        array[i], array[min_loc] = array[min_loc], array[i]
    return array

select_sort([2,3,4,5,9,7,8])



def select_sort(array):
    for i in range(len(array) - 1):
        min_loc = i  # 无序区最小值下标
        for j in range(i, len(array)):
            if array[j] < array[min_loc]:
                min_loc = j
        array[i], array[min_loc] = array[min_loc], array[i]
    return array

select_sort([2,3,4,5,9,7,8])


```





## golang

```Golang
package main

import (
   "fmt"
   "time"
)

//--------------------- Task任务 ---------------------

// 定义一个任务类型 Task
type Task struct {
   f func() error     // 一个Task里面应该有一个具体业务，业务名称叫 f
}

// Task 需要一个执行的方法
func (t *Task) Execute()  {
   t.f()  // 调用任务中已经绑定好的业务方法
}

// 创建一个 Task 任务
func NewTask(arg_f func() error) *Task  {
   t := &Task{
      f: arg_f,
   }
   return t
}

//--------------------- 协程池 ---------------------

// 定义一个pool的协程池
type Pool struct {
   EntryChannel chan *Task // 对外的Task入口 管道
   JobsChannel chan *Task  // 对内的Task任务
   work_num int
}

// 创建 Pool 的函数
func NewPool(num int) *Pool {
   p := &Pool{
      EntryChannel:make(chan *Task),
      JobsChannel:make(chan *Task),
      work_num:num,
   }
   return p
}

// 协程池创建work，让work去工作
func (p *Pool) work(worker_id int)  {
   // 1. 永久从JobsChannel管道中取任务
   // 2. 渠道任务执行任务
   for task := range p.JobsChannel{
      task.Execute()
      fmt.Println("work_id",worker_id,"执行完一个任务")
   }
}

// 让协程池工作
func (p *Pool) run() {
   // 1. 根据 work_num 创建worker工作，每个worker 应该是一个goroutine
   for i:=0;i<=p.work_num;i++{
      go p.work(i)
   }

   // 2. 从EntryChannel取任务，取到任务 发送给 JobsChannel
   for task := range p.EntryChannel{
      p.JobsChannel <- task
   }
}

//--------------------- 主函数 ---------------------

func main()  {
   // 1. 创建一些任务,打印系统时间（之后任务的逻辑就写在这个函数里）
   t := NewTask(func() error {
      fmt.Println(time.Now())
      return nil
   })

   // 2. 创建一个 Pool协程池，最大的worker数量为4
   p := NewPool(4)

   // 3.将任务交给协程池 Pool处理,不断的向p中写入任务t
   go func() {
      for {
         p.EntryChannel <- t

      }
   }()

   // 启动pool，让Pool开始工作，此时Pool会创建worker，让worker工作
   p.run()

}

```











TCP：面向连接的，可靠的，基于字节流的传输层协议。

​	 面向连接的： 首先是一对一才能面向连接，所以TCP不能向UDP那样一个主机同时向多个主机发送同一条消息，一对多是无法做到的。

​     可靠的：无论网络中出现怎么样的网路变化，TCP协议都可以保证报文最终到达接收端。

​     基于字节流：表示我们的消息其实是没有边界的，所以我们消息无论多大都可以进行传输。

​							第二基于字节流表示它是有序的，当前一个字节没有收到，即使后一个字节收到也不能将请求发							送给应用层去处理。



TCP之上是应用层协议，之下是网络层IP协议。



![image-20201205181600658](/Users/shaowei/Library/Application Support/typora-user-images/image-20201205181600658.png)

### 一. TCP

#### 1.1 TCP 的特点

点对点通信：（不能广播 多播），面向连接，只有在连接的情况下才能通信。

双向传递：（全双工）比如：HTTP1.1它是单向的，只能client发送请求，server发送响应。而TCP是可以两端都发送消息。websocket就是把tcp双向传递的全双工特点暴露到websocket应用层中。

字节流：将我们的消息打包成许多个报文段，并且保证有序接收，重复报文接收端可以自动丢弃

​		缺点：不能维护应用报文的边界（例如：HTTP必须自己定义/r/n为结尾，必须自己通过content.lentgh等方式定义body的结尾）

​		优点：不强制应用程序离散的创建数据块，不限制数据块的大小

流量缓冲：client和server端的速度可能是完全不一致的，机器的配置和cpu的配置都是不一致的，这时候他们的处理速度也是不匹配的，这时候通过流量缓冲的功能来解决相应的问题。

可靠的传输服务：保证发送的报文准确的到达对方（通过重发手段控制报文到达）

拥塞控制：

##### 1.1.1 如何标识一个连接

TCP通过TCP四元组来唯一的定义一个链接，四元组包括 源地址，源端口，目标地址，目标端口

所以单主机最大的TCP连接数为2的96次方

但是HTTP3协议中 QUIC协议就是通过 连接ID（Connection ID）来定义的，所以即使 Connection ID被标识完之后 源地址，源端口，目标地址，目标端口发生变化，我们仍然可以复用同一个连接。

![image-20201212211807441](/Users/shaowei/Library/Application Support/typora-user-images/image-20201212211807441.png)

##### 1.1.2 TCP头部报文格式

Source Port 和 Destination Port是来确定一个TCP连接的。

Sequence Number 和 Acknowledgment Number 是唯一标识一个TCP报文， Acknowledgment Number是确认报文，也就是保证数据的可达。

TCP常规的报文长度是20个字节，20个字节之后呢还会有一个TCP Options，这是一个可选的数据，常用的可选选项有如下：



![image-20201212212621335](/Users/shaowei/Library/Application Support/typora-user-images/image-20201212212621335.png)



![image-20201212213906109](/Users/shaowei/Library/Application Support/typora-user-images/image-20201212213906109.png)



#### 1.2 TCP流程

TCP传输中的 SYN 的secquencer number 为什么不是一样的呢？

   因为网络的中的报文会延迟，会复制重发，也可能会丢失，为了对每个报文不产生影响，所以每次的数字都必须是随机的，应该是不同的，以避免此影响。

##### 1.2.1 三次握手中的报文的详细信息

1. SYN报文需要产生一个随机的secquencer number，并将标志位flag中的第7位置为1

2. SYN/ACK 产生一个随机的secquencer number，并将客户端的secquencer number + 1 作为ACK，并将标志位中的第4位ACK和第7位SYN置为1

3. ACK 将报文中ACK置为1，标志位中第4位置为1

   

##### 1.2.2 三次握手的流程状态

1. 起初clent和server都处于ClOSE状态，其中Server需要监听80或443端口，所以它处于LISTEN状态，

![image-20201205224245729](/Users/shaowei/Library/Application Support/typora-user-images/image-20201205224245729.png)

##### 1.2.3 服务端三次握手流程

1. 客户端的SYN网络分组到达服务器端，会把SYN分组插入到SYN队列中，同时服务端发送一个SYN+ACK，这时候服务端用netstat查看状态为SYN-RECEIVED状态。
2. 客户端发送ACK网络分组后，服务端会从SYN队列中移到ACCEPT队列，状态由SYN-RECEIVED状态转为ESTABLISHED状态。
3. 这时候服务端的Nginx应用程序调用accecpt方法将连接从ACCEPT队列里面取出来。

![image-20201205225159338](/Users/shaowei/Library/Application Support/typora-user-images/image-20201205225159338.png)

##### 1.2.4 TCP调优相关

所以SYN和ACCEPT的长度跟我们的负载都是相关的，对于高负载的服务SYN和ACCEPT队列是需要延长的

对于高并发可以延长SYN队列和ACCEPT队列，优化：

​	客户端调用connect方法时候有connect out参数延长，去设置超时时间。



![image-20201205230522894](/Users/shaowei/Library/Application Support/typora-user-images/image-20201205230522894.png)



如何应对SYN攻击？



![image-20201213144428705](/Users/shaowei/Library/Application Support/typora-user-images/image-20201213144428705.png)

##### 1.2.5 TCP 传送 MMS

TCP是一个面向字节流的协议，它不限制应用层传输消息的长度，但是在它之下的网络层和数据链路层由于发送的报文使用的内存是有限的，所以一定会限制报文的长度，所以TCP会将从应用层接收的任意长度的字节流切分成许多个报文段，那么拆分报文段的依据是怎样的呢？是怎么样拆分的呢？这节我们根绝MMS最大报文大小来进行数据报文的拆分



应用层将字节流交给TCP，TCP将字节流拆分为多个segment报文段，那么流分段的依据是什么呢

	1. MMS： 第一个依据MMS最大报文段大小，因为TCP不过不分层IP层一定会分层，IP层分层是没有效率的。
 	2. 流控：例如接收端一次只能接收3个字节，所以发送端只能发送3个字节过来。

![image-20201213150236984](/Users/shaowei/Library/Application Support/typora-user-images/image-20201213150236984.png)

握手阶段如何协商MMS值呢？

​	可以通过类型为2，总长度为4，后面用2个字节告诉接收端我的MMS值大小。



##### 12.6 TCP 重传与确认



TCP如何保证segment段到达对方？ 重传与确认





##### 1.2.7 滑动窗口
















