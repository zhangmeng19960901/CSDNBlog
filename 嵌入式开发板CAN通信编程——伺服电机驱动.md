### 嵌入式开发板CAN通信编程——伺服电机驱动

在实际的嵌入式项目开发过程中，若不涉及上位机与开发板的通信传输数据，那最关键的无非就是两个内容，读取传感器的数据并处理，驱动硬件设备工作。传感器数据的读取内容在前面我已经讲过了，主要就是TTL、RS232、RS485协议的串口编程，我分别给了实例，读取光敏电阻传感器的状态和倾角传感器的实时角度测量信息。那就还有一个下发指令驱动硬件工作的内容，硬件设备的驱动程序一般都由设备厂家完成，集成在设备的驱动器上（关于字符设备驱动程序我之前讲了不涉及硬件操作驱动的程序实现，后面我还会给大家介绍涉及硬件操作的驱动程序实现，并给出实例），我们要做的就是根据设备的驱动通信协议，下发相应的指令即可，这就牵扯到CAN通信编程。这个专题我仍然还是会给出一个实例，供大家学习交流。

#### CAN通信协议

#### 一、首先还是有必要介绍一下CAN通信协议

（1）CAN（Controller Area Network）又称为局域网控制器，汽车上基本都是采用CAN通信，CAN通信协议拥有稳定性和准确性，传输速率比RS485还高。

（2）多主控制：总线空闲时，所有单元都可以发送消息，但当两个以上的单元同时开始发送消息时，总线会根据标识符ID决定优先级。CAN协议是串行异步通信，同一时刻只能有一个发送或接收信息，由CAN_High和CAN_Low两条信号线，以差分信号的形式进行通讯，这一点和RS485是一样的。

（3）CAN通信协议通信速度快，通信距离较远，并且具有错误检测、错误通知与恢复功能。

（4）CAN总线的报文结构：CAN总线上的报文传输有4种不同的帧类型表示和控制，分别为数据帧、远程帧、错误帧、过载帧。我们这里主要介绍数据帧，数据帧携带数据从发送器至接收器。总线上传输的大多是这种帧。从标识符长度上，又可以把数据帧分为标准帧(11 位标识符)和扩展帧(29 位标识符)。数据帧由 7 个不同的位场组成：帧起始、仲裁场、控制场、数据场、CRC 场、应答场、帧结束。其中，数据场的长度为 0~8 个字节。标识符位于仲裁场中，报文接收节点通过标识符进行报文滤波。

（5）Linux系统中CAN总线配置：在Linux系统中，CAN总线接口设备作为网络设备被系统统一管理，在控制台下，CAN总线的配置和以太网的配置使用相同的命令。主要有三个命令：关闭、设置波特率、启动。

```
ifconfig can0 down
ip link set can0 type can bitrate 125000
ifconfig can0 up
//我的开发板中设置的命令是：
canconfig can0 stop
canconfig can0 bitrate 125000
canconfig can0 start
```

#### 二、CAN通信编程中的API

由于Linux系统将CAN设备作为网络设备进行管理，因此在CAN总线应用开发方面，Linux提供了SocketCAN接口，使得CAN总线通信近似于和以太网的通信，应用程序开发接口更加通用，也更加灵活。

（1）初始化：

SocketCAN的初始化与socket网络编程类似，主要包括创建SocketCAN套接字、设置CAN接口名、指定CAN设备、设置CAN通信的地址结构体，绑定套接字和地址结构体。具体如下所示：

```
int socket_fd;
//定义一个socket can通信的地址结构体
struct sockaddr_can addr;
//定义一个ifreq结构体，这个结构体用来配置和获取IP地址、掩码、MTU等接口信息的
struct ifreq ifr;
//创建socketCAN套接字,设置为原始套接字，原始CAN协议
socket_fd = socket(PF_CAN, SOCK_RAW, CAN_RAW);
//设置CAN接口名，对CAN接口进行初始化
strcpy(ifr.ifr_name, "can0");
//指定can0设备
ioctl(socket_fd, SIOCGIFINDEX, &ifr);
//设置CAN协议
addr.can_family = AF_CAN;
addr.can_ifindex = ifr.ifr_ifindex;
//将套接字与can0绑定
bind(socket_fd, (struct sockaddr *)&addr, sizeof(addr));
```

（2）数据发送与接收：

在CAN总线通信数据收发方面，CAN总线与标准套接字稍有不同，比标准套接字更规范，每一次通信都采用can_frame结构体将数据封装成帧。结构体定义如下：

```
struct can_frame{
	canid_t can_id;//can的标识符，帧ID
	_u8 can_dlc;//数据场长度
	_u8 data[8];//数据
};
```

数据发送使用write函数，数据接收使用read函数来实现。具体过程如下：

```
//存放发送和接收报文的can_frame结构体
struct can_frame frame[2] = {{0}};
//发送报文的内容
frame[0].can_id =0x141;
frame[0].can_dlc = 1;
frame[0].data[0] = 0x11;
int nbytes = write(socket_fd, &frame[0], sizeof(frame[0]));
//判断是否发送成功
if(nbytes != sizeof(frame[0])){
	printf("error\n");
}
//接收报文放在frame[1]结构体中
nbytes = read(socket_fd, &frame[1], sizeof(frame[1]));
```

（3）CAN通信API中还包括错误处理和过滤规则设置，这里我们就不赘述了。

#### CAN总线通信的伺服电机

我用的是一款脉塔的伺服电机RMD-X，具体介绍见它的使用说明书。

![ECFF5AF1-2A64-451e-9322-FB30E49FC39F](https://s2.loli.net/2022/05/27/wvhUSmRi2gBT4Fd.png)

它的CAN总线参数及单电机命令手法报文格式为：

![DEDA5055-EDE7-4c26-9FAB-4C237D73782C](https://s2.loli.net/2022/05/27/NzUE6iYQ89wnZ3e.png)

我现在想要伺服电机运行并工作在速度环，通过使用说明书，我找到了电机运动命令：

![9CE05FA9-5AAC-4905-8A3A-A72E24D6BD3B](https://s2.loli.net/2022/05/27/9TqnkSuDfaOJbFm.png)

电机运行命令为1帧，所以它的can_id也就是标识符为0x141，发送启动的报文如下：

```
frame[0].can_id = 0x141;
frame[0].can_dlc = 8;
frame[0].data[0] = 0x88;
frame[0].data[1] = 0x00;
frame[0].data[2] = 0x00;
frame[0].data[3] = 0x00;
frame[0].data[4] = 0x00;
frame[0].data[5] = 0x00;
frame[0].data[6] = 0x00;
frame[0].data[7] = 0x00;
```

伺服电机启动之后想要工作在速度环，就需要发送速度闭环控制指令：

![9B95D5CE-B68D-4db8-A6C7-EE49F8B6A72D](https://s2.loli.net/2022/05/27/Z3YnUBGgJuSjDzb.png)

它的单位0.01dps/LSB，我这里让其运行在360dps/LSB的速度上，标识符和上面一样，那报文数据就是：

```
frame[0].data[0] = 0xA2;
frame[0].data[1] = 0x00;
frame[0].data[2] = 0x00;
frame[0].data[3] = 0x00;
frame[0].data[4] = 0xA0;
frame[0].data[5] = 0x8C;
frame[0].data[6] = 0x00;
frame[0].data[7] = 0x00;
```

有了CAN总线通信的API和伺服电机的CAN总线参数与报文，我们就可以开始写程序了。

#### CAN总线通信编程

其实上面我已经把程序中的内容基本上都讲到了。具体程序不知道你通过上面的内容能不能完整的实现，我相信你是可以的，这里我只把程序流程说一下，具体的实现留给你们，完整代码我会贴在公众号中，要是没能实现再去下载下来看一下。

程序流程：初始化CAN通信套接字——>先发送启动报文——>在while循环中发送速度闭环控制报文——>接收伺服电机反馈回来的数据——>关闭CAN通信套接字。

用到的头文件：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <limits.h>
#include <stdint.h>
#include <net/if.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/uio.h>
#include <linux/can.h>
#include <linux/can/raw.h>
```

在打开开发板时，要先配置CAN通信接口：

![1651912183(1)](https://s2.loli.net/2022/05/27/FdbPS7xZNIK2Muq.png)

之后就可以将编译好的程序在开发板上运行了：启动——>运动。

![1651912240(1)](https://s2.loli.net/2022/05/27/T1BbogmnhvPX6G3.png)

到这里伺服电机就按照你想要的速度运行啦！！

完整代码我会贴在公众号中，需要完整代码的关注公众号回复CAN通信编程获取。有什么问题也可以在下方留言，我看到之后会回复你。

我是河边小乌龟爬，学习嵌入式软件开发路上的一名小学生，欢迎大家相互交流哇。公众号：河边小乌龟爬。

（群名称：嵌入式软件开发交流群；群 号：1004953094）