# 理解字节序


### 1、什么是字节序

字节序，顾名思义就是字节的顺序。更具体的讲，它是多字节数据存储和传输时，字节的顺序。

### 2、为什么有“字节序”这个东西

因为物理内存是以字节为单位进行数据存储，也就是我们常说的计算机基本单位为字节。
因此，单字节的数据（如c或java的char类型的数据）没有字节序这一说，因为获取它只需要读取一个字节。而多字节数据，由于有多个字节，所以在存储和传输可以使用不同的顺序进行操作。

### 3、举个例子

十六进制数 0x12345678 共占4个字节，分别是0x12、0x34、0x56、0x78，因此在该数字中0x12属于高位数据，0x78属于地位数据。

> 注：
> 可以把内存看成是一个很大的数组，4G内存则是一个长度为4294967296的数组。
> 该数组的索引其实就是内存地址，左边是比较小的地址，右边则越来越大，直至最大值。

第一种顺序：低地址存放高位数据，叫**大端模式**
![图片描述](/images/bVbdtmp.png)

第二种顺序：低地址存放低位数据，叫**小端模式**
![图片描述](/images/bVbdtmo.png)

两者，大端模式比较符合人类的阅读习惯；小端模式更符合计算机的处理方式，因为计算机从低位开始处理。

### 4、大小端的应用场景

网络序：也称为网络字节序，**都是大端模式**。因为TCP/IP协议对各层协议统一规定采用大端模式。

主机序：机器的字节序，**有大端模式也有小端模式**，根据具体机器的处理决定的，小端模式较常见。

那么，在进行网络通讯时，在将本地数据发送到网络上就需要注意这个顺序。如果本地机器的字节序是小端模式，则需要先转换成大端模式后再进行发送；如果本地已经是大端模式，则可以直接发送。

ps.在私有协议上的数据，只要收发两端约定好，用什么顺序都行。不过，还是建议用网络序发送。

### 5、如何判断主机序

```c
int main() {
    int x = 0x1020304;
    char* p = (char*)&x;
    if(p[0]==1){
         printf("Big\n");
    }
    else{
          printf("Little\n");
    }
    return 0;
}
```

### 6、Java设置大小端

```java
public class HelloEndian {

    public static void main(String[] args) {
        ByteBuffer b = ByteBuffer.wrap(new byte[4]);
        b.order(ByteOrder.BIG_ENDIAN);
        b.putInt(0x01020304);
        System.out.println("Big-Endian:    " + Arrays.toString(b.array()));

        b = ByteBuffer.wrap(new byte[4]);
        b.order(ByteOrder.LITTLE_ENDIAN);
        b.putInt(0x01020304);
        System.out.println("Little-Endian: " + Arrays.toString(b.array()));
    }

}
```
