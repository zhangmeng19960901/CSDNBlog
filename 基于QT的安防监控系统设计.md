## 基于QT的安防监控系统设计

#### 包含的知识点：

开发板：开发板的内核，设备节点，服务器网络通信程序，多线程编程，Linux操作系统的基本操作，文件系统挂载。

摄像头：v4l2的常用函数，lcd操作，Framebuffer，

QT：Qt的基本使用，信号与槽，界面设计，与开发板的通信，QUdpSocket常用函数。

#### 项目开发流程：

预期目标：桌面APP能够实时查看摄像头采集到的画面。并能捕捉某一时刻的画面。

整体实现方式：桌面APP实现与ARM开发板通信，开发板读取摄像头采集的数据，并保存在内核，APP通过QUdpSocket与开发板建立连接，ARM开发板接收APP的指令信息之后（recvfrom函数）通过UDP网络通信将采集到的图像画面传输给APP上显示（sendto）。

APP界面基于Qt来开发。

开发板采集摄像头数据基于v4l2视频处理模块。

#### 开发板采集摄像头数据并发送数据给App(服务器程序)：

##### 使用的开发板相关知识：

使用的linux开发板是GEC6818开发平台，搭载三星Cortex-A53（CPU架构）系列高性能八核处理器S5P6818。支持嵌入式Linux和Android操作系统的驱动、应用开发。

STM32MP157开发板内核架构是Cortex-A7和M4，支持Linux操作系统开发和单片机开发。

##### 采集摄像头数据涉及的知识点：LCD画图，Framebuffer模块。

Framebuffer模块：在Linux系统中通过Framebuffer驱动程序来控制LCD，Freambuffer就是一块内存，里面保存着一帧图像

简单介绍LCD的操作原理：

① 驱动程序设置好LCD控制器：

根据LCD的参数设置LCD控制器的时序、信号极性；

根据LCD分辨率、BPP分配Framebuffer。

② APP使用ioctl获得LCD分辨率、BPP

③ APP通过mmap映射Framebuffer，在Framebuffer中写入数据

##### v4l2视频处理模块：常用的函数

v4l2介绍：video for Linux的简称，是Linux视频处理模块的最新标准代码，相机捕捉到光线通过视频芯片处理，编码成JPG/MJPG/YUV格式输出，通过v4l2与相机进行通信，设置或获取他们的参数。

```
linux_v4l2_device_init("/dev/video7");//利用v4l2初始化摄像头设备；
linux_v4l2_start_capturing();//启动摄像头
linus_v4l2_get_frame(&framebuf);//获取摄像头数据，并存放在freambuf中
lcd_dram_jpg(80, 0, BULL, framebuf.buf, framebuf.length, 0);//在LCD上显示图像
linux_v4l2_stop_capturing();//停止摄像头
linux_v4l2_device_uinit();//卸载摄像头
```

##### 开发板Udp网络通信传输数据：

```
int Socket_fd = socket(AF_INET, SOCK_DGRAM, 0);//指定协议族AF_INET/PF_INET,SOCK_DGRAM/SOCK_STREAM使用数据包传输还是字节流传输
struct sockaddr_in address;//IPv4的地址结构体，之后给地址结构体赋值
address.sin_family = AF_INET;//指定使用的地址族
address.sin_port = htons(port);//指定感兴趣的端口号，并要转换成网络字节序
inet_pton(AF_INET, ip, &address.sin_addr);//将ip地址转换成网络字节序，赋值给sin_addr
int ret = bind(Socket_fd, (struct sockaddr*)&address, sizeof(address));//将ip地址和感兴趣的port关联到创建的udp传输描述符
ret = listen(Socket_fd, backlog);//监听是否有连接，backlog为连接的最大数
//这里的recvfrom()和sendto是Udp传输专用的API函数。
recvfrom(Socket_fd, buf, sizeof(buf), 0, (struct sockaddr*)&send_addr, &addrlen);//将接收到数据存放在buf中， 并将对方的ip地址和port端口号存储在send_addr结构体中。
sendto(Socket_fd, buf, sizeof(buf), 0, (struct sockaddr*)&send_addr, &addrlen);//将数据buf发送给地址和端口号为send_addr的机器上。
//TCP数据读写函数
ssize_t recv(socketfd, buf, sizeof(buf), 0);
ssize_t send(socketfd, buf, sizeof(buf), 0);
//接收连接与发起连接函数
int accept(socketfd, struct sockaddr* addr, socklen_t *addrlen);//接收连接保存的是对方的ip和port
int connect(socketfd, const struct sockaddr* addr, socklen_t *addrlen);//服务器的ip和端口号
```

##### 多线程编程实现：

实现程序通过使用多线程编程，开发板运行的主线程接收应用程序发送来的消息，创建另一条线程去执行实时监控并发送数据给应用程序。

```
pthread_t tid;//创建一个线程描述符
pthread_create(&tid, NULL, Jpd_real_time, NULL);//创建一个线程执行函数
void *Jpg_Real_Time(void *arg){
	线程具体实现的功能；
};//线程关联函数的定义与实现
```

#### 监控应用程序接收开发板图像并显示：

##### Qt相关知识点：界面设计、信号与槽。

界面设计：目前主要用到了Push Button、Line Edit、Text Edit、Label等设计模块

```
ui->lineEdit/textEdit/label->setText(str);//将str显示在相应的模块上
ui->pushButton->text();//读取标签上的内容
```

信号与槽：将信号和槽函数关联在一起

```
QPushButton *push_num_button[] = {
        ui->pushButton_1,
        ui->pushButton_2,};	//定义一组按键的数组
connect(push_num_button[i], SIGNAL(clicked(bool)), this, SLOT(槽函数))；//将信号与槽函数绑定在一起。本句中的意思是将按键pushButton1与槽函数关联，通过clicked即点击触发
QPushButton *QBt = qobject_cast<QPushButton *>(sender());//获取信号发送的控件指针
```

网络通信涉及的函数：

```
QUdpSocket();//进行UDP通信的函数，需要先添加头文件和路径
Recv_Socket = new QUdpSocket();//定义一个文件描述符
connect(Recv_socket,SIGNAL(readyRead()), this, SLOT(Recv_Message()));//网络通信
按键pushBotton可以直接转到槽，自动生成槽函数。
Recv_Socket->readDatagram(buf, sizeof(buf), &ip, &port);//接收信息存储到buf中，并将对方的ip地址和端口号保存下来；
Recv_Socket->writeDatagram(buf, ip, port);//将数据buf发送到ip地址与端口号port的主机上
```

