### 嵌入式项目实战——基于QT的视频监控系统设计（三）

进入到五一假期第三天，继续我们的项目。本来五一假期还是想好好休息一下的，因为最近学习的状态不太好，刷题都没有思路了，但是身边的同学太卷了，不过我还是想放松一下，所以上午睡觉，下午复盘一下这个项目分享出来。等假期结束之后，再好好冲刺一波。

前两天分别介绍了QT的基本使用以及UDP网络编程，实现了用QT编写一个上位机与开发板进行数据传输。这些工作完成之后我们就可以开始关注在上位机中显示视频画面了，这里面涉及到开发板内核的视频画面获取与处理，然后通过UDP网络通信发送给上位机实时显示。

#### 第三天：v4l2视频处理模块

首先介绍一下v4l2视频处理模块，V4L2是V4L的第二版，是Video For Linux的简称，V4L早在Linux 2.1中就已经被引入，到2.6.38时才有了V4L2的最终替代。V4L2是Linux视频处理模块的最新标准代码，包括对视频输入设备的处理，如高频(即、电视信号输入端子)或摄像头，还包括视频处理输出装置。一般来说，最常见的是使用V4L2来处理相机数据采集的问题。我们通常使用的相机实际上是一个图像传感器，将捕捉到的光线通过视频芯片处理后，编码成JPG/MJPG或YUV格式输出。我们可以很容易地通过V4L2与第一台摄像机设备“通信”，如设置或获取它们的工作参数。

在内核中,摄像头捕捉到的视频数据,我们可以使用一个队列来存储。我们做的工作大致是这样的：首先配置摄像头的相关参数，可以正常工作，然后申请一个号码的内核视频缓存，送他们到队列，像三个空盘子在传送带上。然后我们还需要三个内核缓存区域通过mmap函数映射到用户空间，这样我们就可以操作相机数据在用户层，然后我们可以启动相机开始数据采集，每一帧捕获数据我们可以做一个团队操作，读取数据，然后再阅读内核缓存的数据小组，依次循环。

![image-20220502151024812](https://s2.loli.net/2022/05/27/X4D8e37m1wzKLJq.png)

下面是v4l2视频处理模块常用的函数：

```
linux_v4l2_device_init("/dev/video7");//利用v4l2初始化摄像头设备；
linux_v4l2_start_capturing();//启动摄像头
linus_v4l2_get_frame(&framebuf);//获取摄像头数据，并存放在freambuf中
linux_v4l2_stop_capturing();//停止摄像头
linux_v4l2_device_uinit();//卸载摄像头
```

这里又出现了一个重要的知识点，Framebuffer模块，简单介绍一些Framebuffer模块：Freambuffer简单来说就是一块内核中的内存，里面保存着一帧图像。

掌握这里知识点，我们就可以实现在上位机中实时显示摄像头拍摄的画面啦。

上位机我写的上位机程序里已经完成了显示图象的代码：

```
//头文件中声明
private:
    QPixmap pix;
//主函数中显示
//QPixmap类显示图片
pix.loadFromData((uchar *)data, ret);
pix = pix.scaled(ui->label->width(), ui->label->height());
ui->label->setPixmap(pix);
```

开发板中运行的服务器程序可以在上一节中介绍的Server.c程序中修改，我这里采用的是使用多线程编程，用一个线程来实现实时监控功能

```
FrameBuffer freambuf;//声明一个FrameBuffer结构体，存放视频数据
void *Jpg_Real_Time(void *arg)	//实时监控功能代码
{
	/* 初始化摄像头设备*/
	linux_v4l2_device_init("/dev/video7");	
	/* 启动摄像头*/
	linux_v4l2_start_capturing();	
	while(1)
	{
		// 实时监控
		/* 获取摄像头数据      存放jpg文件流*/
		linux_v4l2_get_fream(&freambuf);			
		/* 显示摄像头图像*/
		lcd_draw_jpg(80, 0, NULL, freambuf.buf, freambuf.length, 0);
		/* WiFi发送摄像头图像*/
		sendto(Socket_fd, freambuf.buf, freambuf.length, 0, (struct sockaddr *)&Phone_ipaddr, addrlen);
	}	
	/* 停止摄像头*/
	linux_v4l2_stop_capturing();	
	/* 卸载摄像头*/
	linux_v4l2_device_uinit();
}
```

初始化UDP通信的网络接口

```
const char* ip = "192.168.5.100";
void Udp_Init()	//创建UDP套接字。Bind指定的IP和端口号
{
	Socket_fd = socket(AF_INET, SOCK_DGRAM, 0);
	if(Socket_fd < 0)
	{
		perror("socket failed");
		return ;
	}
	Arm_ipaddr.sin_family = AF_INET;
	Arm_ipaddr.sin_port   = htons(2234);
	inet_pton(AF_INET, ip, &Arm_ipaddr.sin_addr);	
	addrlen = sizeof(struct sockaddr_in);
	int ret = bind(Socket_fd, (struct sockaddr *)&Arm_ipaddr, addrlen);
	if(ret < 0)
	{
		perror("bind failed");
		return ;
	}	
}
```

在主线程中需要完成的就是接收上位机发送过来的信息，并解析保存上位机的IP地址和端口号，线程Jpg_Real_Time( ）得到上位机的IP地址和端口号之后，就可以将采集到视频数据通过UDP传输给上位机显示。

```
int main(int argc,char**argv)
{
	//0,初始化udp网络通信
	Udp_Init();
	//1,创建一条线程去实现实时监控
	pthread_t tid;
	pthread_create(&tid, NULL, Jpg_Real_Time, NULL);
	//2,主线程接受手机端发送的数据
	char buf[128];
	char ip_addr[20] = {0};
	while(1)
	{
		bzero(buf, sizeof(buf));
		recvfrom(Socket_fd, buf, sizeof(buf), 0,(struct sockaddr *)&Phone_ipaddr, &addrlen);
		printf("from:%s port:%d recvbuf:%s\n", inet_ntop(AF_INET, &Phone_ipaddr.sin_addr, ip_addr, sizeof(ip_addr)), ntohs(Phone_ipaddr.sin_port), buf);	
	}
	return 0;
}
```

到这里就全部完成了所有的程序设计，下面就需要在Ubuntu中编译要在开发板中运行的服务器程序了。使用如下命令编译程序。因为我们需要显示的是jpeg格式的图片数据，所以需要在编译文件路径下包含jpeg解码包。如果你用的YUYV格式的摄像头，还需要将采集到的图片转换成jepg格式的图片才能显示。

```
arm-linux-gnueabi-gcc realtime_video.c -o realtime_video -I ./libjpeg -L ./libjpeg -lapi_v4l2_arm -lpthread -ljpeg
```

解释一下上面的内容

```
-I./libjpeg ： 指定动态库头文件位置
-L./libjpeg ： 指定动态库，库文件位置
-ljpeg ： 指定动态库名
```

之后在开发板上插入摄像头，运行程序，打开上位机，点击摄像头按钮，就可以显示摄像头采集的数据了

![1651490083(1)](https://s2.loli.net/2022/05/27/TIHehvFA492xQLr.png)

至此这个项目需要掌握的知识点基本上算是讲完了，后面无非就是对上位机的美化与重新设计了，牵扯到的还是网络编程，来发送数据的问题。

完整代码我会贴在公众号中（河边小乌龟爬），需要完整代码的关注公众号回复视频监控第三讲获取。有什么问题也可以在下方留言，我看到之后会回复你。

我是河边小乌龟爬，学习嵌入式软件开发路上的一名小学生，欢迎大家相互交流哇。公众号：河边小乌龟爬。

（群名称：嵌入式软件开发交流群；群 号：1004953094）