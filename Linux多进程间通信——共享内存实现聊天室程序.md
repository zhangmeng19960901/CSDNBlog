### Linux多进程间通信——共享内存实现聊天室程序

上一讲我们用共享内存实现了进程间的简单通信，一个进程写，一个进程读，我们这次增加一点难度，创建两块共享内存，来实现一个简单的聊天室程序。

先自己动手做一下，再来看源码。

下面给出源码，一共有四个进程，两两实现通信，互相收发消息

shmmutexRW_Server.c代码如下：

```
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/sem.h>
#include <sys/shm.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <unistd.h>
int main(){
	sem_t *sem_1;//定义信号量
	sem_t *sem_2;
	int shmid[2];//共享内存ID

	struct Student{
		char name[20];
		int age;
		int number;
	};
	struct Student *shmaddr[2];
	//创建信号量来实现读写锁，读写不能同时进行
	sem_1 = sem_open("sem_1", O_CREAT, 0644, 1);
	sem_2 = sem_open("sem_2", O_CREAT, 0644, 1);
	
	//创建子进程
	pid_t pid;
	pid = fork();
	
	if(pid < 0){
		perror("cannot fork!\n");
		return -1;
	}
	else if(pid == 0){
		//子进程，用共享内存shmaddr[0]来接收shmMutexRW_Server.c子进程的消息
		key_t shmkey;
		shmkey = ftok("/etc", 0);//创建共享内存的标识
		shmid[0] = shmget(shmkey, 100, 0666|IPC_CREAT);
		if(shmid[0] == -1){
			printf("shmget failed\n");
			exit(0);
		}
		shmaddr[0] = (struct Student*)shmat(shmid[0], 0, 0);
		if(shmaddr[0] == (struct Student*)-1){
			printf("shmat failed\n");
			exit(0);
		}
		//开始读此进程中的数据并显示
		shmaddr[0]->age = 0;
		while(1){
			sem_wait(sem_1);
			if(shmaddr[0]->age > 0){
				printf("子进程Server读到共享内存中的数据：姓名：%s\n", shmaddr[0]->name);
				shmaddr[0]->age--;
			}
			sem_post(sem_1);
		}
		sem_close(sem_1);
		if(shmdt(shmaddr[0]) == -1) printf("shmdt failed\n");
		if(shmctl(shmid[0], IPC_RMID, NULL) == -1) printf("shmctl failed\n");	
	}
	else{
		//父进程,向共享内存shmaddr[1]中写入数据，供shmMutexRW_Server.c的父进程读数据
		//创建共享内存shmaddr[1]
		key_t shmkey1 = ftok("/bin", 0);
		shmid[1] = shmget(shmkey1, 100, 0666|IPC_CREAT);
		if(shmid[1] == -1){
			printf("shmget failed\n");
			exit(0);
		}
		shmaddr[1] = (struct Student*)shmat(shmid[1], 0, 0);
		if(shmaddr[1] == (struct Student*)-1){
			printf("shmat failed\n");
			exit(0);
		}
		shmaddr[1]->age = 0;//这里很关键，还是信号量的赋初值很关键？
		while(1){
			sem_wait(sem_2);
			memset(shmaddr[1]->name,  0, 20);	
			printf("写进程：请输入姓名（不超过20字符）到共享内存：\n");
			if(fgets(shmaddr[1]->name, 20, stdin) == NULL){
				perror("fgets failed\n");
				sem_post(sem_2);
				break;
			}
			shmaddr[1]->age++;
			sem_post(sem_2);
			sleep(1);
		}
		sem_close(sem_2);
		if(shmdt(shmaddr[1]) == -1){
			printf("shmdt failed\n");
		}
		if(shmctl(shmid[1], IPC_RMID, NULL)){
			printf("shmctl failed\n");
		}	
	}
	sem_unlink("shmaddr[0]_RW");
	sem_unlink("shmaddr[1]_RW");
	return 0;
}
```

这里shmmutexRW_Client.c的源码我就不再给出了，读者们可以根据上一讲的内容来自己实现一下。

程序运行的效果如下：

![CE86BC2C-395D-4d06-964A-76CA5F90B427](https://s2.loli.net/2022/05/27/KZx5SJ8Wgf9Iy7G.png)

完整源码我放在公众号河边小乌龟爬中了，需要的自取，回复共享内存聊天室即可获得。

我是河边小乌龟爬，学习嵌入式软件开发路上的一名小学生，欢迎大家相互交流哇。公众号：河边小乌龟爬。