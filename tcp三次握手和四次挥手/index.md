# tcp三次握手和四次挥手

> 人，只要有一种信念，有所追求，什么艰苦都能忍受，什么环境也都能适应。——丁玲

### 三次握手流程图

![1002211-20191130222422452-127219771](/images/1002211-20191130222422452-127219771.png)

### 四次挥手流程图

![u=958516549,2180335224&fm=26&gp=0](/images/u=958516549,2180335224&fm=26&gp=0.jpg)



### tcp报文

![2](/images/2.png)

其中，Seq(Sequence Number) 是 32 位的序列号，`Acknowledgment number` 称之为确认序号，也是 32 位的，但是他不是标志位 ACK，这个要区别开，当 ACK 置 1 时 `Acknowledgment number` 才有效，指接收方期待的下一个报文段的序列号。

TCP 报头信息中有六个控制位(标志位)，分别是：SYN、ACK、PSH、FIN、RST 和 URG。

```
SYN: 表示建立连接
FIN: 表示关闭连接
ACK: 表示响应
PSH: 表示有数据传输
RST: 表示连接重置
URG: 表示紧急数据
```

Seq 序列号有两个作用：
第一，在 SYN 置 1 时，此为当前连接的初始序列号(Initial Sequence Number, ISN)该值是个随机值，数据的第一个字节序号为此 ISN+1。

第二，在 SYN 置 0 时，为当前连接报文段的累计数据包字节数。

### 网络模型

![1629935-20190412155007282-2076867106](/images/1629935-20190412155007282-2076867106.png)

计算机网络分层模型主要有OSI七层模型，TCP/IP五层模型，也有一种四层模型，四层模型会把网卡层和物理层统称为网络接口层。

OSI七层模型存在于教科书了，TCP/IP五层模型是日常运用最为广泛的一种网络架构模型。在学习网络知识时也要把握住重点去学，七层模型了解即可。

![130957m7v2xta7kzft2d7v](/images/130957m7v2xta7kzft2d7v.jpg)

### 代码准备

server.php

```
<?php
//创建Server对象，监听 127.0.0.1:9501端口
$serv = new Swoole\Server("127.0.0.1", 9501);

//监听连接进入事件
$serv->on('Connect', function ($serv, $fd) {
    echo "Client: Connect.\n";
});

//监听数据接收事件
$serv->on('Receive', function ($serv, $fd, $from_id, $data) {
    $serv->send($fd, "Server: ".$data);
});

//监听连接关闭事件
$serv->on('Close', function ($serv, $fd) {
    echo "Client: Close.\n";
});

//启动服务器
$serv->start();
```

client.php

```
<?php
//client.php
$client = new swoole_client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
$client->connect('127.0.0.1', 9501) || exit("connect failed. Error: {$client->errCode}\n");

// 向服务端发送数据
$client->send("Just a test.\r\n");
$client->close();
```

### wireshark准备

抓包网络选择lookback

筛选条件输入ip.addr == 127.0.0.1 and  tcp.port in {9501}

### Tcp请求

直接请求client.php

```
php  client.php
```

### 抓包

### 模型分析

![image-20200523184857040](/images/image-20200523184857040.png)

上面模型网络里有五层，这里只有四层，是因为我直接请求的tcp，没有走应用层，所以只有四层。

### 三次握手抓包分析

看到wireshark已经抓到了请求

![image-20200523180836466](/images/image-20200523180836466.png)

##### 第一次握手

客户端向服务器发送连接请求包，标志位SYN=1，序列号Seq=0（为方便下面的计算用这里的Seq起名为X），如下图

![image-20200523182423880](/images/image-20200523182423880.png)

![132355zqm2skuwm8q2yuzg](/images/132355zqm2skuwm8q2yuzg.jpg)

##### 第二次握手

服务器收到客户端发过来报文，由SYN=1知道客户端要求建立联机。向客户端发送一个SYN和ACK都为1的TCP报文，设置初始序号Seq=0（为方便下面的计算用这里的Seq起名为Y），将确认序号(Acknowledgement Number)设置为客户的序列号加1，即X+1=0+1=1，所以这里的序列号为1, 如下图

![image-20200523182738543](/images/image-20200523182738543.png)

![132450t4spsguuwlaemez5](/images/132450t4spsguuwlaemez5.jpg)

##### 第三次握手

客户端收到服务器发来的包后检查确认序号(Acknowledgement Number)是否正确，即第一次发送的序号加1（X+1=1）。以及标志位ACK是否为1。若正确，客户端再次发送确认包，ACK标志位为1，SYN标志位为0。确认序号(Acknowledgement Number)=Y+1=0+1=1，发送序号为X+1=1。服务端收到后确认序号值与ACK=1则连接建立成功，可以传送数据了。

![image-20200523183640836](/images/image-20200523183640836.png)

![132538pqssss4ceszes4sx](/images/132538pqssss4ceszes4sx.jpg)

### 四次挥手抓包分析

![image-20200523220651612](/images/image-20200523220651612.png)

##### 第一次挥手

客户端给服务器发送TCP包，用来关闭客户端到服务器的数据传送。将标志位FIN和ACK置为1，序号为X=15，确认序号为Z=1。

![image-20200523221619960](/images/image-20200523221619960.png)

![132803xomlsmpbzmrioibb](/images/132803xomlsmpbzmrioibb.jpg)

##### 第二次挥手

服务器收到FIN后，发回一个ACK(标志位ACK=1),确认序号为收到的序号加1，即X=X+1=16。序号为收到的确认序号=Z。

![image-20200523221933721](/images/image-20200523221933721.png)

![132951z9uzba8ind90n8ai](/images/132951z9uzba8ind90n8ai.jpg)

##### 第三次挥手

服务器关闭与客户端的连接，发送一个FIN。标志位FIN和ACK置为1，序号为Y=1，确认序号为X=16。

![image-20200523222145938](/images/image-20200523222145938.png)

![133044hlr71pqqqurr7pzc](/images/133044hlr71pqqqurr7pzc.jpg)

##### 第四次握手

客户端收到服务器发送的FIN之后，发回ACK确认(标志位ACK=1),确认序号为收到的序号加1，即Y+1=2。序号为收到的确认序号X=16。

![image-20200523222333500](/images/image-20200523222333500.png)

![133145oqqlxqgxoaz1g13w](/images/133145oqqlxqgxoaz1g13w.jpg)


