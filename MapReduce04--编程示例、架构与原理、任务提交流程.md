# 
## 编程模型
- 借鉴函数式的编程方式
- 用户只需要实现两个函数接口：
```
Map(in_key, in_value)
    -> (out_key, intermediate_value) list
Reduce (out_key, intermediate_value list)
    ->out_value list
```
## 编程示例
### WordCount实现过程
源数据
- Document 1    ：  the weather is good
- Document 2    ：  today is good
- Document 3    ：  good weather is good

### 我的第一个MapReduce任务

![我的第一个MapReduce任务](DD539130A894469DB7D900F4A42DE41F)  

- Map 输出
  - Worker 1  ：  (the 1), (weather 1), (is 1), (good 1)
  - Worker 2  ：  (today 1), (is 1), (good 1)
  - Worker 3  ：  (good 1), (weather 1), (is 1), (good 1)  


- map输出的是key、value对。例子中的“1”指的就是value。意思是这个key出现了一次。  
最后reduce阶段进行的操作是将value的“1”相加。的得到的就是key出现的次数。


- Shuffling过程就是将map阶段输出的相同的key进行整理。  
按照key进行取模，然后找到key对应的reduce。


- Reduce 输入
  - Worker 1  ：  (the 1)
  - Worker 2  ：  (is 1), (is 1), (is 1)
  - Worker 3  ：  (weather 1), (weather 1)
  - Worker 4  ：  (today 1)
  - Worker 5  ：  (good 1), (good 1), (good 1), (good 1)

- Reduce 输出
  - Worker 1  ：  (the 1)
  - Worker 2  ：  (is 3)
  - Worker 3  ：  (weather 2)
  - Worker 4  ：  (today 1)
  - Worker 5  ：  (good 4)

## MapReduce实现架构
### 两个重要的进程（Hadoop1.0）
- JobTracker
  - 主进程，负责接收客户作业提交，调度任务到作节点上运行，并提供诸如监控工作节点状态及任务进度等管理功能。  
    一个MapReduce集群有一个jobtracker，一般运行在可靠的硬件上。
  - tasktracker是通过周期性的心跳来通知jobtracker其当前的健康状态，每一次心跳包含了可用的map和reduce任务数目、占用的数目以及运行中的任务详细信息。  
    Jobtracker利用一个线程池来同时处理心跳和客户请求。
- TaskTracker
  - 由jobtracker指派任务，实例化用户程序，在本地执行任务并周期性地向jobtracker汇报状态。  
    在每一个工作节点上永远只会有一个tasktracker
  - 如果TaskTracker出现问题（网络传输、机器负载等）不能胜任工作，要把信息通报给JobTracker。
  - 如果工作中超过一段时间JobTracker没有得到丛节点的汇报，JobTracker就会认为TaskTracker出现了问题，JobTracker会把相同的任务分发给其他的TaskTracker  
 
![image](82BF731CEBFC4350AD71BA8A4C14C75B)  

## MapReduce工作原理
- JobTracker一直在等待JobClient提交作业
- TaskTracker每隔3秒向JobTracker发送心跳询问有没有任务可做。
  - 如果有任务，让其派发任务给它执行。
  - 如果TaskTracker正在做工作，他也会通过心跳的方式告JobTracker其当前的工作做到什么程度了。
- Slave主动向master拉生意


![image](F4F2D5E0941148D0984886AAE669D0DB)  

## MapReduce任务提交流程

1. 用户是工程师提交的作业，作业是在工程师自己机器上提交的。机器就是一个客户端，即JobClient。

2. JobClient在请求一次任务时，要与JobTracker通信。想要让JobTracker分配任务。
> 如果JobTracker当时能够执行任务，就会就收JobClient的请求。（TaskTracker会定期向JobTracker发送心跳，以告诉JobTracker他现在是否处于空闲状态）  
如果TaskTracker无空闲，那么JobClient就会排队。（或者被拒绝）

3. JobTracker接受JobClient请求后，会反馈给JobClient一个Job  id号。
> 这个id就代表着JobClient提交的任务，用来帮JobClient找到他提交的任务。

5. JobClient拿到Job id后，会在HDFS上创建一个目录。这个目录的名字就叫做Job   id。
> 这个Job   id相当于JobClient与JobTracker的一个约定。

6. 把用户写的代码或程序从本地拷贝到这个HDFS的目录里。

7. 通知JobTracker，已经将代码或程序传到HDFS上。然后让JobTracker去执行这个作业。

8. JobTracker会进行Job初始化。然后就开始分配工作。

9. JobTracker会把HDFS上的代码分发给TaskTracker（这些代码就相当于执行任务的要求）。

10. TaskTracker执行任务时会去HDFS上找Job    id号所对应的代码。然后开始执行任务

11. TaskTracker会定期向JobTracker发送心跳，以汇报其当前执行情况。

12. 每个TaskTracker输入的数据源是不同的（分配给TaskTracker的Split不同）。
> JonClient要求读的数据源是在HDFS上，用户写的代码也是在HDFS上。数据不是放在Job id目录下，而是分布在整个集群不同的block里。  
所以，TaskTracker不仅要读Job id上的代码，还要给TaskTracker指定数据路径。


## MapReduce作业调度

- MapReduce作业会有一个优先级
> 每个用户提交作业时，都会分配一个优先级。  
任务执行时会在要执行的任务里找到优先级高的任务，先执行。

- 默认先进先出队列调度模式（FIFO）  
优先级（very_high、high、normal、low、very low）
