### 嵌入式项目实战——基于QT的视频监控系统设计（一）

这个五一因为疫情，只能待在家里，想了想不如将我之前做的一个小的嵌入式的练习项目分享出来，供入门嵌入式的同学们学习。基于QT的视频监控系统设计虽然是个小项目，但是涉及的嵌入式的知识点还是比较多的，比如多线程编程，网络编程，QT界面设计，LCD显示，如果有时间我也会介绍一下触摸屏的使用，v4l2视频解码以及嵌入式开发板的基本操作。下面就开始我们五一假期的学习吧！！

#### 第一天：QT的基本使用和UDP网络编程

#### 一、QT的基本使用——完成一个简易的随机选餐软件设计

Qt Creator是一个用于Qt开发的轻量级跨平台集成开发环境。Qt并不是一门编程语言。Qt是一门用标准C++编写的跨平台开发类库，它对标准C++进行了拓展，引入了元对象系统，信号与槽，属性等特性，使应用程序的开发变得更加高效。控件就是Qt提供的一种图形类，在设计编程时，可以直接拖入设计界面(mainwindow.ui)进行编程。QT最关键的使用就是信号与槽的使用，槽即是一个函数，将要实现的功能、运算在槽函数里面定义清楚，之后通过图形界面上的按钮信号触发槽函数，前提是要先将按钮信号与槽函数绑定在一起。理解了这个就掌握了QT编程的基础，再熟悉一点QT的基本语法之后，就能开发一个QT软件啦。

下面就跟着我开始你的第一个QT界面软件设计吧！！

首先安装你的QT Creator软件，下载地址我也贴在这里，省的你们去找了（https://download.qt.io/archive/qt/5.9/5.9.1/）。下载之后跟着一步步安装即可，我就不一步一步的安装贴在上面了，毕竟我们的重点是使用撒。提示：安装组件一定要选择MinGW 5.3.0，如果你打算开发Android软件的话，还需要勾选Android ARMv7。

![image-20220430163320792](https://s2.loli.net/2022/05/27/j5M8OgIL3rExiyB.png)

安装完成之后，打开桌面的图标即可进入软件界面。

![image-20220430164039895](https://s2.loli.net/2022/05/27/uGkNetdwJfhOEIH.png)

之后进入下一步，命名你的项目名称，选择你要保存的路径，然后点击下一步。

![image-20220430164450105](https://s2.loli.net/2022/05/27/Qy1eiDOLkdZM5RF.png)

接下来你要选择你的编译工具，它决定了你是创建一个桌面软件还是Android软件，这里我还没有配置Android开发环境，所以只有一个选项，如果你想开发一个Android软件，需要提前配置环境，可以留言，我会出一期这方面的教程。

![image-20220430164841314](https://s2.loli.net/2022/05/27/OytQFXxhfWGzCsw.png)

之后一直点击下一步，直到完成创建项目。出现下面这些就表明项目已经创建成功。你就可以在这个项目框架上大展拳脚啦。

![image-20220430165014690](https://s2.loli.net/2022/05/27/geIsFmnMGRi3bxf.png)

我们今天要设计的是一款随机选餐软件，我选择这个奇怪的题目的原因是我对象每天吃饭的时候都发愁纠结不知道吃哪一个，她就说让我给她做一个选餐软件，免得她纠结，今天正好介绍QT，把这个软件做一下。这是一个功能很简单的软件哦。跟着做下去你一定能够实现的。

首先设计我们的界面。双击Froms->mainwindow.ui。

![image-20220430165821066](https://s2.loli.net/2022/05/27/ynLUBfKX4HWe7Gc.png)

进入界面设计，我们首先点击画布空白区域，在右下角找到window Title，给软件命名。

![image-20220430170149189](https://s2.loli.net/2022/05/27/9FiHuOZmNLfosSR.png)

下面开始你的界面设计，我设计的这个界面只有两个按键，两个显示条，成品就是下面这个展示的这个，你可以根据你的偏好设计属于你的界面。按键：Push Button，显示条：Line Edit。效果就是点击按键《点击这里开始选餐》，显示条就会随机显示一个餐品名，如果不满意，点击按键《最后一次选择机会》，会重新在显示条随机显示一个餐品名。

![image-20220430192020653](https://s2.loli.net/2022/05/27/eT1WJdsOiAIbBtw.png)

下面就开始最关键的内容了，开始设计代码，关联信号与槽函数了，QT有两种关联方式。一种是直接鼠标左键点击按键，选择转到槽，再选择信号类型，点击OK，就会自动生成槽函数。

![image-20220430192722294](https://s2.loli.net/2022/05/27/mWYC2BbhGnQyz3D.png)

在mainwindow.cpp中就会自动生成一个槽函数

```
void Mainwindow::on_pushButton_clicked(){

}
```

在mainwindow.h头文件中也会自动声明一个槽函数

```
private slots:
    void on_pushButton_clicked();
```

![image-20220430192910997](https://s2.loli.net/2022/05/27/2oPNhcXvVq6lzg9.png)

还有一种关联按键信号和槽函数的方式，这种方式适合一次将多个信号关联到一个槽函数

```
QPushButton *push_num_button[] = {
        ui->pushButton_1,
        ui->pushButton_2,};	//定义一组按键的数组
connect(push_num_button[i], SIGNAL(clicked(bool)), this, SLOT(槽函数))；//将信号与槽函数绑定在一起。本句中的意思是将按键pushButton1与槽函数关联，通过clicked即点击触发
```

本软件因为是两个按键，我就一个用系统自动生成，一个用connect函数来关联。用connect()函数来关联按键《最后一次选择机会》的代码如下。

![image-20220430202815564](https://s2.loli.net/2022/05/27/EPaFTwNX76fI4qd.png)

在mainwindow.h中同样需要声明一个参函数。

```
private slots:
    void on_pushButton_clicked();
    void on_pushButton_2_clicked();
```

现在我们需要在槽函数里面实现我们的功能了。我们需要实现什么功能呢，很简单，就是当我们点击《点击这里开始选餐》按键（这里对应的信号与槽函数是pushButton，on_pushButton_clicked()）时，会在显示条line_Edit_2上显示随机出现的餐名，点击《最后一次选择机会》按键（这里对应的信号与槽函数时pushButton_2，on_pushButton_2_clicked()）时，会在显示条line_Edit_2上重新显示一个随机的餐名。现在功能已经明确了，那我们就开始动手实现吧！

要想实现随机选餐功能，我们就需要生成一个随机数

```
#include <ctime>//头文件
qsrand(time(NULL));
int n = qrand() % 5;//生成一个5以内的随机数
```

之后就变得很简单啦，我们先定义一些字符串，然后比较生成的随机数是什么，选择相应的字符串显示在显示条中就可以啦！！

```
QString s1 = "五谷渔粉";
QString s2 = "凉面凉皮";
QString s3 = "关东煮";
QString s4 = "恰一碗素汤粉";
QString s5 = "辣椒鸡丝刀削面";
```

这里我用的是最简单的方式实现的，直接定义多个QString，也可以用字符串列表来实现，代码更简洁一些。这个就留给你们自己去探索吧。

然后我们在对应的槽函数里来处理按键按下之后的逻辑就可以了

```
void MainWindow::on_pushButton_clicked()
{
    qsrand(time(NULL));
    int n = qrand() % 5;//生成一个5以内的随机数
    QString tmp;
    if(n == 0){
        tmp = s1;
    }
    else if(n == 1){
        tmp = s2;
    }
    else if(n == 2){
        tmp = s3;
    }
    else if(n == 3){
        tmp = s4;
    }
    else if(n == 4){
        tmp = s5;
    }
    ui->lineEdit_2->setText(tmp);//将字符串tmp显示在line_Edit_2显示条上
}
```

点击左下角的编译按钮就可以生成软件啦

![image-20220430204658750](https://s2.loli.net/2022/05/27/GuYKdvO9FWTDnkM.png)

到这里这个随机选餐的软件就完成啦，是不是很容易上手啊，下面是成品展示效果。

![ezgif.com-gif-maker](https://s2.loli.net/2022/05/27/cib4CFKPELdtYTk.gif)

本来今天打算把网络编程的内容也一并讲了的，写这个博客花费了太多时间，明天在补上吧。

完整代码我会贴在公众号中，需要完整代码的关注公众号回复随机选餐软件获取。也什么问题也可以在下方留言，我看到之后会回复你。

我是河边小乌龟爬，学习嵌入式软件开发路上的一名小学生，欢迎大家相互交流哇。公众号：河边小乌龟爬。

（群名称：嵌入式软件开发交流群；群 号：1004953094）