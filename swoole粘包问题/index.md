# swoole粘包问题

> 自知之明是最难得的知识。

### TCP传输的特点

TCP是一种面向连接的、可靠的、基于字节流的传输层通信协议

#### 面向连接

面向连接意味着使用tcp的应用程序在传输数据前必须先建立连接，三次握手

#### 基于字节流的传输

发送端执行的写操作数和接收端执行的读操作次数之间没有任何数量关系，当发送端应用程序连续执行多次写操作的时，TCP模块先将这些数据放入TCP发送缓冲区中。当TCP模块真正开始发送数据的时候，发送缓冲区中这些等待发送的数据可能被封装成一个或多个TCP报文段发出

#### 可靠

- TCP协议采用发送应答机制，即发送端发送的每个TCP报文段都必须得到接收方的应答，才能认为这个TCP报文段传输成功。
- TCP协议采用超时重传机制，发送端在发送出一个TCP报文段之后启动定时器，如果在定时时间内未收到应答，它将重新发送该报文段。

### 什么是TCP粘包问题

TCP粘包是一个错误的说法，TCP是流式传输，没有包的概念，更没有粘包一说。

人们常说的粘包问题，是指发送方应该以什么格式发送数据，接收方能正确解析出数据，这个叫应用层协议，你自己定，跟TCP完全无关。如果是发文件，最简单的你可以用http协议封装，如果你发的http协议数据是100%正确的，无论哪个接收方（nginx/tomcat/iis）保证能一字节不差地收下，因为http协议本身就带header和body，header里有Content-Length: 12345指定了body的大小，body才是文件本身。

你不用http协议，直接发文件数据，那么问题来了，接收方怎么知道应该收多少字节后文件结束？这个才是真正粘包问题的由来。

所有TCP粘包问题就转化成了如何界定TCP流式传输边界的问题。

### TCP粘包的原因

#### 缓冲区

通常我们直觉性的认为，客户端直接向网络中传输数据，对端从网络中读取数据，但是这是不正确的。

socket有缓冲区buffer的概念，每个TCP socket在内核中都有一个发送缓冲区和一个接收缓冲区。客户端send操作仅仅是把数据拷贝到buffer中，也就是说send完成了，数据并不代表已经发送到服务端了，之后才由TCP协议从buffer中发送到服务端。此时服务端的接收缓冲区被TCP缓存网络上来的数据，而后server才从buffer中读取数据。

```
当连续发送数据时，由于tcp协议的nagle算法，会将较小的内容拼接成大的内容，一次性发送到服务器端，因此造成粘包
当发送内容较大时，由于服务器端的recv（buffer_size）方法中的buffer_size较小，不能一次性完全接收全部内容，因此在下一次请求到达时，接收的内容依然是上一次没有完全接收完的内容，因此造成粘包现象。
接收数据端的应用层没有及时读取接收缓冲区中的数据，将发生粘包。
```

![1503450-20200318205534350-396366488](/images/1503450-20200318205534350-396366488.png)

#### 粘包、拆包表现形式

第一种情况，接收端正常收到两个数据包，即没有发生拆包和粘包的现象。

第二种情况，接收端只收到一个数据包，由于TCP是不会出现丢包的，所以这一个数据包中包含了发送端发送的两个数据包的信息，这种现象即为粘包。这种情况由于接收端不知道这两个数据包的界限，所以对于接收端来说很难处理。

第三种情况，这种情况有两种表现形式。接收端收到了两个数据包，但是这两个数据包要么是不完整的，要么就是多出来一块，这种情况即发生了拆包和粘包。这两种情况如果不加特殊处理，对于接收端同样是不好处理的。

### TCP粘包的解决方法

- EOF 结束符协议
- 固定包头 + 包体协议

#### 粘包复现

创建一个server，server端代码如下

```php
<?php

class TcpBufferServer
{
    private $_serv;

    /**
     * init
     */
    public function __construct()
    {
        $this->_serv = new Swoole\Server("127.0.0.1", 9501);
        $this->_serv->set([
            'worker_num' => 1,
        ]);
        $this->_serv->on('Receive', [$this, 'onReceive']);
    }
    public function onReceive($serv, $fd, $fromId, $data)
    {
        echo "Server received data: {$data}" . PHP_EOL;
    }
    /**
     * start server
     */
    public function start()
    {
        $this->_serv->start();
    }

}

$reload = new TcpBufferServer;
$reload->start();
```

server的代码很简单，仅仅是在收到客户端代码后，标准输出一句话而已，client的代码需要注意了，我们写了一个for循环，连续向server send三条信息，代码如下

```php
<?php

$client = new swoole_client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
$client->connect('127.0.0.1', 9501) || exit("connect failed. Error: {$client->errCode}\n");

// 向服务端发送数据
for ($i = 0; $i < 3; $i++) {
    $client->send("Just a test.\n");
}
$client->close();
```


在未运行测试的情况下，我们期望server所在终端输出的结果应该是这样的

```php
<?php
Server received data: Just a test.
Server received data: Just a test.
Server received data: Just a test.
```


注意哦，我们期望的结果是server被回调了3次，才有上述期望的结果值

实际运行的结果呢，是否与我们所期望的一致？我们看下

![img](/images/c484586336-tcp-buffer-1.png)

上图左边是server输出的信息。

我们看到，左侧显示的结果是server一次性输出的结果，按理论来说，client发起了3次请求，server应该跟我们期望的结果一致，会执行3次呀，但是事实并不是，这就产生了粘包，看下抓包的结果。

![image-20200524160856253](/images/image-20200524160856253.png)

![image-20200524161040797](/images/image-20200524161040797.png)

由抓包的图可见，三次请求的内容到了缓冲区后，作为两次发送，有两个数据发生了粘包，到了服务端缓冲区后，是三个一起取出产生了粘包。

#### EOF结束协议

EOF，end of file，意思是我们在每一个数据包的结尾加一个eof标记，表示这就是一个完整的数据包，但是如果你的数据本身含有EOF标记，那就会造成收到的数据包不完整，所以开启EOF支持后，应避免数据中含有EOF标记。

在swoole_server中，我们可以配置open_eof_check为true，打开EOF检测，配置package_eof来指定EOF标记。

swoole_server收到一个数据包时，会检测数据包的结尾是否是我们设置的EOF标记，如果不是就会一直拼接数据包，直到超出buffer或者超时才会终止，一旦认定是一个完整的数据包，就会投递给Worker进程，这时候我们才可以在回调内处理数据。

这样server就能保证接收到一个完整的数据包了？不能保证，这样只能保证server能收到一个或者多个完整的数据包。

为啥是多个呢？

我们说了开启EOF检测，即open_eof_check设置为true，server只会检测数据包的末尾是否有EOF标记，如果向我们开篇的案例连发3个EOF的数据，server可能还是会一次性收到，这样我们只能在回调内对数据包进行拆分处理。

我们拿开篇的案例为例

server开启eof检测并指定eof标记是\r\n，代码如下

```php
<?php

class TcpBufferServer
{
    private $_serv;

    /**
     * init
     */
    public function __construct()
    {
        $this->_serv = new Swoole\Server("127.0.0.1", 9501);
       /* $this->_serv->set([
            'worker_num' => 1,
        ]);*/
        $this->_serv->set([
            'worker_num' => 1,
            'open_eof_check' => true, //打开EOF检测
            'package_eof' => "\r\n", //设置EOF
        ]);
    
        $this->_serv->on('Receive', [$this, 'onReceive']);
    }
    public function onReceive($serv, $fd, $fromId, $data)
    {
        echo "Server received data: {$data}" . PHP_EOL;
    }
    /**
     * start server
     */
    public function start()
    {
        $this->_serv->start();
    }

}

$reload = new TcpBufferServer;
$reload->start();
```

客户端设置发送的数据末尾是\r\n符号，代码如下

```php
<?php

$client = new swoole_client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
$client->connect('127.0.0.1', 9501) || exit("connect failed. Error: {$client->errCode}\n");

// 向服务端发送数据
/*for ($i = 0; $i < 3; $i++) {
    $client->send("Just a test.\n");
}*/

for ($i = 0; $i < 3; $i++) {
    $client->send("Just a test.\r\n");
}

$client->close();
```


按照我们刚才的分析，server的效果可能会一次性收到多个完整的包，我们运行看看结果

![img](/images/845ca5d229-server-eof-check.png)

明显不是我们想要的结果，抓包的结果与上面是一样的。

原来我们还需要在onReceive回调内对收到的数据进行拆分处理

```php
<?php
public function onReceive($serv, $fd, $fromId, $data)
{
    // echo "Server received data: {$data}" . PHP_EOL;

    $datas = explode("\r\n", $data);
    foreach ($datas as $data)
    {
        if(!$data)
            continue;
    
        echo "Server received data: {$data}" . PHP_EOL;
    }

}
```


此时我们再看下运行结果

![img](/images/6ff4cf8e12-server-eof-check-2.png)

自行分包的效果便实现了，考虑到自行分包稍微麻烦，swoole提供了open_eof_split配置参数，启用该参数后，server会从左到右对数据进行逐字节对比，查找数据中的EOF标记进行分包，效果跟我们刚刚自行拆包是一样的，性能较差。

在案例的基础上我们看看open_eof_split配置

```php
<?php
$this->_serv->set([
    'worker_num' => 1,
    'open_eof_check' => true, //打开EOF检测
    'package_eof' => "\r\n", //设置EOF
    'open_eof_split' => true,
]);
```

onReceive的回调，我们不需要自行拆包

```php
<?php
public function onReceive($serv, $fd, $fromId, $data)
{
     echo "Server received data: {$data}" . PHP_EOL;
}
```


client的测试代码使用\r\n（同server端package_eof标记一致），我们看下运行效果

![img](/images/af126fe46e-server-eof-split.png)

EOF标记解决粘包就说这么多，下面我们再看看另一种解决方案

#### 固定包头+包体协议

固定包头是一种非常通用的协议，它的含义就是在你要发送的数据包的前面，添加一段信息，这段信息了包含了你要发送的数据包的长度，长度一般是2个或者4个字节的整数。

在这种协议下，我们的数据包的组成就是包头+包体。其中包头就是包体长度的二进制形式。比如我们本来想向服务端发送一段数据 "Just a test." 共12个字符，现在我们要发送的数据就应该是这样的

```php
<?php
pack('N', strlen("Just a test.")) . "Just a test."
```

其中php的pack函数是把数据打包成二进制字符串。

为什么这样就能保证Worker进程收到的是一个完整的数据包呢？我来解释一下：

当server收到一个数据包（可能是多个完整的数据包）之后，会先解出包头指定的数据长度，然后按照这个长度取出后面的数据，如果一次性收到多个数据包，依次循环，如此就能保证Worker进程可以一次性收到一个完整的数据包。

我们以案例来分析

server代码

```php
<?php

class ServerPack
{
    private $_serv;

    /**
     * init
     */
    public function __construct()
    {
        $this->_serv = new Swoole\Server("127.0.0.1", 9501);
        $this->_serv->set([
            'worker_num' => 1,
            'open_length_check'     => true,      // 开启协议解析
            'package_length_type'   => 'N',     // 长度字段的类型
            'package_length_offset' => 0,       //第几个字节是包长度的值
            'package_body_offset'   => 4,       //第几个字节开始计算长度
            'package_max_length'    => 81920,  //协议最大长度
        ]);
        $this->_serv->on('Receive', [$this, 'onReceive']);
    }
    public function onReceive($serv, $fd, $fromId, $data)
    {
        $info = unpack('N', $data);
        $len = $info[1];
        $body = substr($data, - $len);
        echo "server received data: {$body}\n";
    }
    /**
     * start server
     */
    public function start()
    {
        $this->_serv->start();
    }

}

$reload = new ServerPack;
$reload->start();
```

客户端的代码

```php
<?php

$client = new swoole_client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_SYNC);
$client->connect('127.0.0.1', 9501) || exit("connect failed. Error: {$client->errCode}\n");

// 向服务端发送数据
for ($i = 0; $i < 3; $i++) {
    $data = "Just a test.";
    $data = pack('N', strlen($data)) . $data;
    $client->send($data);
}

$client->close();
```

运行的结果

![img](/images/4f422b928f-server-pack.png)

结果没错，是我们期望的结果。

我们来分析下这是为什么

1、首先，在server端我们配置了open_length_check，该参数表明我们要开启固定包头协议解析

2、package_length_type配置，表明包头长度的类型，这个类型跟客户端使用pack打包包头的类型一致，一般设置为N或者n，N表示4个字节，n表示2个字节

3、我们看下客户端的代码 pack('N', strlen($data)) . 𝑑𝑎𝑡𝑎，这句话就是包头+包体的意思，包头是𝑝𝑎𝑐𝑘函数打包的二进制数据，内容便是真实数据的长度𝑠𝑡𝑟𝑙𝑒𝑛(data，这句话就是包头+包体的意思，包头是pack函数打包的二进制数据，内容便是真实数据的长度strlen(data)。

在内存中，整数一般占用4个字节，所以我们看到，在这段数据中0-4字节表示的是包头，剩余的就是真实的数据。但是server不知道呀，怎么告诉server这一事实呢？

看配置package_length_offset和package_body_offset，前者就是告诉server，从第几个字节开始是长度，后者就是从第几个字节开始计算长度。

4、既然如此，我们就可以在onReceive回调对数据解包，然后从包头中取出包体长度，再从接收到的数据中截取真正的包体。

```php
<?php
$info = unpack('N', $data);
$len = $info[1];
$body = substr($data, - $len);
echo "server received data: {$body}\n";
```


