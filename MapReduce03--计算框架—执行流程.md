# MapReduce计算框架-执行流程
## 图一

![单级程序计算流程](CA9102F206EE4CC6ACCF4CEDACCC79BA)  
> 应用程序开发人员一般情况下需要关心的是图中灰色的部分!

1. 数据存储在HDFS上，HDFS将数据散落在整个集群的各个机器上。 

2. 数据从HDFS出来后经过InputFormat接口，把大的数据切分成小的split（分片）。  
> 这个接口是MR实现的一个类，在这个类里提供了两种功能：  
（1）数据分割（Data Splits）  
（2）记录读取器（Record Reader）  

3. 切完后的split作为map的输入。  

4. 数据在map中进行逻辑处理，然后通过shuffle&sort再其传递给reduce。  
> 不同reduce内的key是不同的。

5. reduce进行逻辑处理后，输出结果到output。   

## 图二

![image](83DB081C91014B439AF06AD97646AC6D)  

### map阶段

1. 将split分片的数据读入map，一个split分片对应一个map。
2. 因为map是一个程序，所以会在计算机系统内启动一个进程。然后会自己维护一个进程空间。
把split数据读入后，相当于直接读入到内存中（buffer in memory ）
3. split中的数据开始写入内存中，因为map维护进程空间的内存是有大小的（默认是100M）。所以需要往磁盘中转储数据。  
> (1)当写到其内存的80%时，将80%已经写好的内存锁住;  
  (2)将80%内存的数据统一往磁盘中dump（转储），并清空内存。即把内存中的数据转储到磁盘中;  
  (3)转储的过程不仅仅是转移数据，还有一次sort（排序）。

4. 在数据写入到内存的过程中，会有多次的转储过程，因此会在磁盘中产生很多小的文件（partition sort and spill to disk）；
数据全部写完后，通过算法（类似归并排序）将所有的小文件合并成一个大的文件（大文件也是排好序的）。
> 因为每个小的文件都是经过sort的，所以可以用类似于**归并排序里的merge()函数**的方法合并小文件。这样得到的大文件也是排好序的。

### reduce阶段

1. Reduce个数可以自己设置；  
2. 把每个map上相同的key放在一起，拷贝到一个redece上；  
3. 拷贝后小文件很多，需要经过多次合并后得到大的文件再给reduce；  
4. reduce做数据计数工作，最后输出。  

## 图三

![image](4512DA8716044768890DA716BF085397)  

1. 把数据输入到内存区（缓冲区），当内存区满了后，将数据往磁盘转储。
2. 转储的过程需要排序，默认为快速排序。排序是根据key来排。排序是为了把相同的key排到一起。
3. 转储到磁盘的小文件命名为Spill.n。

## 图四

![image](235F3FDA9C0C4182BCC0EBE72B3ADBAE)  

1. 小文件Spill经过**多次归并排序**后得到一个大文件。
2. 相同的key为一个partition。

## 图五

![image](FE77E94401744F89B68830449F5D7B9B)  

1. 图五左侧，每一个map里都可以分为几个partition。
> 在map中是将相同的key放到一个partition中。也可以将多个key放到一个partition中（要保证在partition外无与partition内相同的key）
2. 进入reduce阶段，将每个map里相同的partition拷贝到一个reduce中。
3. 通过merge()函数合并partition。然后发送给reduce。
4. reduce也是一个进程。
5. reduce进行逻辑处理完后输出。


