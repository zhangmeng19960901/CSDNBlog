### Linux多进程间通信——套接字实现

前面我们分享了进程间通信的一种方式——共享内存，现在我们来讲实现不同主机之间的进程间通信方式，其实这个问题我之前就讲过，这里再给大家总结一下。

下面就是UDP/TCP网络编程中常用的API接口函数：

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

![E5BDDCBB-D45C-4346-B838-B2C41255F811](https://s2.loli.net/2022/05/27/XR6HNGlos7upa3j.png)

更多嵌入式相关内容，请关注公众号河边小乌龟爬。有什么问题可以在下方留言，我看到之后会回复你。

我是河边小乌龟爬，学习嵌入式软件开发路上的一名小学生，欢迎大家相互交流哇。公众号：河边小乌龟爬。