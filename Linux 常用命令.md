# Linux 常用命令

#### change：改变	directory：目录	list：列出	print：打印	make：创建

#### remove：删除	copy：复制	move：移动	clear：清除

#### -r：recursive，递归地，即所有文件

#### -f：force，强制删除

很多命令都是从这些单词中简化而来的

### 1.创建文件和文件夹命令

mkdir -p(路径)  -m(权限)  目录名

touch 文件名

rmdir 目录名（为空）

rm -r 目录名

rm 文件名

### 2.改变工作路径

cd

### 3.显示当前目录的绝对路径

pwd

### 4.列出目录的内容

ls  -l(显示文件的详细信息)

ls -al(显示所有文件的包括隐藏文件的详细信息)

ls  -c(按照修改时间排序)

### 5.改变文件的访问权限

chmod ++(增加权限)  --(减小权限)

chmod 777 文件，可读可写可执行，r, w, x

### 6.查找相关内容

grep -rn "查找内容"  [查找路径]	r：递归查找	n：显示行号

find  是查找文件	find /home/zhm/dir/  -name  "*.text"	在目录下以名字来查找.text后缀的文件

### 7.文件复制与移动

cp -r/a(赋值目录时)  源文件  目标目录

mv  源文件  目标目录

### 8.显示文件内容

cat 文件名

### 9.挂载文件系统

mount -t nfs -o nolock,vers=3 192.168.5.11:/home/zhm/nfs_rootfs  /mnt

mount  /dev/mmcblk2p2   /boot      挂载驱动系统

### 10.驱动相关的命令

insmod （-f（强制安装）） 驱动文件.ko  加载安装驱动

rmmod  驱动文件  卸载驱动

lsmod  已经加载的驱动

ls  /dev/   显示所有的设备节点

cat  /proc/devices  显示所有的设备号

### 11.查找函数使用用法及应包含的头文件

man -f 函数名	查看函数属于哪一类

man  数字(哪一类)   函数	查看函数的使用方法

### 12.进程相关linux命令

./应用程序	& 	后台运行

ps	查看进程号及进程名称

kill 进程号	结束进程

### 13.从普通用户进入root管理员环境与退出

sudu su

exit

### 14.网络命令

ifconfig：查看当前正在使用的网卡

ifconfig -a：查看所有网卡

sudo ifconfig ens160(eth0) 192.168.1.137：设置静态IP

udhcpc -i eth0	尝试再次获取IP

### 15.vi相关指令

vi 文件名	打开文件

a/i	进入编辑模式

esc按键	进入命令行模式

:wq	保存并退出

:w	保存

:wq	保存并退出

:q!	强制退出

shift +zz	保存并退出

Ctrl + f	屏幕向下翻一页	Ctrl + b	向上翻一页

ndd	删除当前行向下n行

nyy	复制当前行及向下n行

p	粘贴最近复制的内容

### 16.查看文件类型

file 文件名	,查看文件的详细信息

### 17.安装更新软件

sudo apt-get update/upgrade

sudo apt-get install

### 18. 压缩与解压

压缩：tar  -zcvf  test.tar.gz  test.txt	gzip test.txt

解压：tar  -zxvf  test.tar.gz				  gunzip test.txt.gz

###  19.Ubuntu与开发板互传文件

scp -r aaaa.c root@192.168.1.99

