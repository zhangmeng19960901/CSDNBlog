### 嵌入式项目实战——基于QT的视频监控系统设计（二）

昨天我分享了关于QT的基本使用方法，掌握了这些基本的方法就可以设计一个简单的视频监控界面。下面我们开始分享完成这个嵌入式项目同样重要的知识点——UDP网络编程，网络编程是实现上位机与开发板通信的重要手段，在TCP/IP协议族中TCP/UDP是传输层实现数据传输的两种方式，TCP传输层协议在传输数据时需要先建立TCP连接，连接建立需要三次握手，关闭连接又需要四次挥手，TCP协议是基于字节流的传输协议，是一种稳定可靠的传输方式，但是它不适合于视频传输，视频传输由于传输数据量较大，需要源源不断的将数据传输到客户端，但由于网络的不确定性，将会导致数据丢失，这时只有使用不可靠的UDP传输才能保证视频的流畅性，因为你丢了数据，UDP协议不需要服务器重新发送，你只管将下一帧发给我就可以，UDP是基于数据报的传输层协议，且可以一对多、多对多传输。

下面我们就开始今天的学习并设计一个简单的QT界面实现与开发板的通信，显示一些开发板服务器发送过来的数据。

#### 第二天：UDP网络编程和实现与开发板通信

#### 一、UDP网络编程

TCP/IP协议族中包括五个主要部分：应用层（DNS）、传输层（TCP/UDP）、网络层（IP/ICMP）、数据链路层与硬件层（ARP/RARP）。我们本节只介绍UDP协议，UDP协议为应用层提供不可靠、无连接和基于数据报的服务，“不可靠”意味着UDP协议无法保证数据准确的从发送端到达目的端，如果数据丢失，发送端不重传数据。“无连接”意味着通信双方不保持一个长连接，因此应用程序每次发送数据时都要明确指定接收端的IP地址和端口信息。基于数据报的服务时相对于TCP协议基于字节流的服务而言的，每个UDP数据报都有一个长度，接收端必须以该长度为最小单位将其内容一次性读出，否则数据将被截断。这就UDP协议的基本知识点，面试的时候可能会问。

下面就是UDP网络编程中常用的API接口函数：

```
int Socket_fd = socket(AF_INET, SOCK_DGRAM, 0);//指定协议族AF_INET/PF_INET,SOCK_DGRAM/SOCK_STREAM使用数据包传输还是字节流传输
struct sockaddr_in address;//IPv4的地址结构体，之后给地址结构体赋值
address.sin_family = AF_INET;//指定使用的地址族
address.sin_port = htons(port);//指定感兴趣的端口号，并要转换成网络字节序
inet_pton(AF_INET, ip, &address.sin_addr);//将ip地址转换成网络字节序，赋值给sin_addr
int ret = bind(Socket_fd, (struct sockaddr*)&address, sizeof(address));//将ip地址和感兴趣的port关联到创建的udp文件描述符
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

有了上面的API函数我们就可以实现一个客户端程序和一个服务器程序，并在Ubuntu中实现客户端程序和服务器程序的通信。

客户端程序：

```
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <pthread.h>
#include <strings.h>
#include <sys/socket.h>
#include <netinet/in.h>	
#include <arpa/inet.h>
#include <pthread.h>
int main(){
	int Socket_fd;//UDP套接字
	struct sockaddr_in client_ipaddr, server_ipaddr;//声明两个UDP协议地址结构体
	const char* ip = "192.168.20.128";//指定UDP通信的IP地址和端口号
	int port = 6666;
	Socket_fd = socket(PF_INET, SOCK_DGRAM, 0);//指定使用的协议族和数据传输方式，这里的传输方式是数据报传输，即UDP传输
	if(Socket_fd < 0){
		perror("socket failed!");
	}	
	//下面是服务器地址协议结构体中的内容
	server_ipaddr.sin_family = AF_INET;
	server_ipaddr.sin_port = htons(port);
	inet_pton(AF_INET, ip, &server_ipaddr.sin_addr);
	//向指定的IP地址和端口号的主机发送一条信息
	char sendbuf[100] = "an importment message!";
	sendto(Socket_fd, sendbuf, sizeof(sendbuf), 0, (struct sockaddr*)&server_ipaddr, sizeof(server_ipaddr));
}
```

接收端程序：

```
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <pthread.h>
#include <strings.h>
#include <sys/socket.h>
#include <netinet/in.h>	
#include <arpa/inet.h>
#include <pthread.h>

int main(){
	int Socket_fd;//UDP套接字
	struct sockaddr_in client_ipaddr, server_ipaddr;//声明两个UDP协议地址结构体
	const char* ip = "192.168.20.128";//指定UDP通信的IP地址和端口号
	int port = 6666;
	Socket_fd = socket(PF_INET, SOCK_DGRAM, 0);
	if(Socket_fd < 0){
		perror("socket failed!");
	}	
	//下面是服务器地址协议结构体中的内容
	server_ipaddr.sin_family = AF_INET;
	server_ipaddr.sin_port = htons(port);
	inet_pton(AF_INET, ip, &server_ipaddr.sin_addr);
	int ret = bind(Socket_fd, (struct sockaddr*)&server_ipaddr, sizeof(server_ipaddr));//将ip地址和感兴趣的port关联到创建的udp文件描述符
	char buf[128];
	socklen_t client_ipaddrlen = sizeof(client_ipaddr);
	while(1)
	{
		bzero(buf, sizeof(buf));
		recvfrom(Socket_fd, buf, sizeof(buf), 0,(struct sockaddr *)&client_ipaddr, &client_ipaddrlen);
		printf("recvbuf:%s\n", buf);	
	}
}
```

在Ubuntu中分别编译这两个程序：

![image-20220501170453578](https://s2.loli.net/2022/05/27/t8KGSxjRrIC5mOd.png)

首先运行接收端程序，会一直等待数据发送过来

![image-20220501170650651](https://s2.loli.net/2022/05/27/WtclNJapiw9VudK.png)

然后运行客户端程序

![image-20220501170745713](https://s2.loli.net/2022/05/27/6PHulFTcXtKJQLe.png)

接收端会接收到数据：

![image-20220501170821744](https://s2.loli.net/2022/05/27/JyU5BtCxZP4kHpR.png)

到这里UDP网络编程的内容就完成了。

#### 二、实现上位机与开发板通信

结合第一天讲的QT的基本使用方法，我相信你能很容易的设计下面这个界面，这个界面实现的功能就是点击摄像头按键，会在TextLabel显示框中显示开发板传回的信息。

![image-20220501184953354](https://s2.loli.net/2022/05/27/xHO768pRAlhPqbU.png)

现在我们已经在QT中完成了上位机的UI界面设计，下一步就是要实现上位机与开发板的通信了。在QT中封装了专用的UDP传输的类：QUdpSocket。QUdpSocket类常用的函数如下：

```
QUdpSocket();//进行UDP通信的函数，需要先添加头文件和路径
Recv_Socket = new QUdpSocket();//定义一个文件描述符
```

网络通信按键pushBotton可以直接转到槽，自动生成槽函数。

```
connect(Recv_socket,SIGNAL(readyRead()), this, SLOT(Recv_Message()));
```

接收信息存储到buf中，并将对方的ip地址和端口号保存下来

```
Recv_Socket->readDatagram(buf, sizeof(buf), &ip, &port);
```

将数据buf发送到ip地址与端口号port的主机上

```
Recv_Socket->writeDatagram(buf, ip, port);
```

有了这些函数就能实现上位机和开发板的通信了，这里我就不把全部的代码贴出来了，完整代码我会放在公众号中，回复视频监控第二讲即可获取。

下面就是写一个在开发板中运行的服务器程序了。可以在上面的UDP_Server.c的基础上进行修改即可，要实现的功能就是在收到上位机发送过来的数据之后，得到上位机的IP和端口号，进而发送一个信息给上位机显示。

下面开始演示整个实验过程。

首先登陆开发板，我习惯用MobaXterm来登录开发板，设置开发板的静态IP，这个IP地址要是我们服务器程序和上位机程序保持一致的IP，我这里设置为192.168.5.100

![image-20220501191143317](https://s2.loli.net/2022/05/27/3fzb6CEk4nPdvtI.png)

我习惯于挂载Ubuntu的文件系统到开发板，实现开发版与Ubuntu文件互传。

![image-20220501191437958](https://s2.loli.net/2022/05/27/8eua43Qw7pZnLCU.png)

之后在Ubuntu上用交叉编译工具编译要在开发板上运行的服务器程序Server.c

![image-20220501191752618](https://s2.loli.net/2022/05/27/1Dr58WpxEJtT6HZ.png)

现在就可以在开发板上运行了。

![image-20220501192312443](https://s2.loli.net/2022/05/27/lnhuOEQaMGXbDpV.png)

然后点击上位机中的摄像头按键。开发板收到连接指令，将发送一个信息给上位机并显示。

![image-20220501192525484](https://s2.loli.net/2022/05/27/p1aKkZtcQCUqwV3.png)

到这里本节的内容就基本上讲完了，实现了上位机与开发板的通信，我们将利用这些知识点最终实现上位机实时显示摄像头拍摄的画面。

完整代码我会贴在公众号中，需要完整代码的关注公众号回复视频监控第二讲获取。有什么问题也可以在下方留言，我看到之后会回复你。

我是河边小乌龟爬，学习嵌入式软件开发路上的一名小学生，欢迎大家相互交流哇。公众号：河边小乌龟爬。

（群名称：嵌入式软件开发交流群；群 号：1004953094）