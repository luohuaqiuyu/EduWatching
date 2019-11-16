[toc]

# 项目简介

本项目基于PaddlePaddle和EasyDL平台，以教务处和学工为一级用户，高校教师为二级用户，针对提升整体课堂教学质量为目的开发的一款实时课堂监测系统。

本项目主要监测课堂的出勤人数、学生的上课状态、教师的语速、情感，以及语言的用词方面。项目中语音的模型均采用**EasyDL平台**进行训练，调用在线API进行预测分析。而图像模型由于在线API无法达到实时性的要求，采用本地训练**Paddle模型库**中的模型并使用。

## 硬件环境

CPU：Intel 酷睿 I7-7700 四核8线程

内存：三星 DDR4 16G

GPU：NVIDIA GTX1070 8G

## 软件环境

OS： Windows 10

IDE：PyCharm 2019.2.4

ffmpeg（需要加入环境变量）

Python 3.7

CUDA10 CUDNN7.3 

### Python依赖

baidu_aip==2.2.18.0
jieba==0.39
opencv_python==4.1.1.26
requests==2.22.0
PyMySQL==0.9.3
paddlepaddle_gpu==1.6.0.post107
numpy==1.16.5
Pillow==6.2.0
PyQt5==5.10.1

# 模型详解

## EasyDL平台模型

EasyDL平台的快速训练和快速上线是目前人工智能开发进程中的一大亮点，能够作为项目中的一个在线API进行快速调用。但是在线调用非常受网速限制，对于图片这种体积较大的文件则更加耗时，在实时性方面有待提高。但是本地部署需要企业帐号，对于一部分开发者来说无法实现。如果能将模型下载到本地进行类似SDK的方式调用，将会更好。

| 模型ID | 模型功能                                     |
| ------ | -------------------------------------------- |
| 41825  | 识别学生上课的状态                           |
| 26127  | 检测教室内的所有学生                         |
| 26080  | 检测教室内不同状态的学生（直接分类物体检测） |
| 26109  | 检测教室内不同状态的学生（删掉模糊的标注）   |
| 24516  | 识别教师的讲课的情感（12分类）               |
| 25450  | 识别教师的讲课语速（10分类）                 |
| 42835  | 识别教师的讲课情感（4分类）                  |

## YOLOv3目标检测

项目中教室内学生的位置检测以及人数统计使用Paddle模型库中的YOLOv3模型

### 数据集

采用我校教务处提供的一周（5天）教学视频为基础，每天视频时长14小时（8：00——22：00，有前后两个摄像头），每隔10分钟截取一张图片，一共289张图片，进行人为手工标准（EasyDL平台上也有相同模型）

![](/readmeimg/标注.jpg)

### 训练

在本机上以batch_size=2，一共训练20000轮

![](/readmeimg/A_front_34.png)

## RestNet图像分类

在目标检测将学生位置识别出来之后，将这些学生图像单独抠出来放入到图像分类模型中进行分类，分为：正常听课、看手机、睡觉、站立，采用Paddle模型库中的ResNet模型，56层

### 数据集

在上一步目标检测标注完的数据集基础上，将所有学生抠出来保存成单独的图片，进行分类，一共正常：2431张，看手机：1149张，睡觉：189张，站立：56张

为了不让数据样本偏差太大，随机抽取比较平衡的数据量：正常：300张，看手机：300张，睡觉：189张，站立：56张

![A_front_0_14](/readmeimg/A_front_0_14.jpg)![A_front_3_5](/readmeimg/A_front_3_5.jpg)![A_front_3_11](/readmeimg/A_front_3_11.jpg)![A_front_289_0](/readmeimg/A_front_289_0.jpg)

### 训练

以batch_size=8，进行100轮的训练

top1 acc=0.7

### 调用方式

## 情感分类

使用Paddle模型库中Senta情感分类模型进行文字的情感倾向分析

### 数据集

从互联网中查找在课堂场景中的语言文本，分为积极、消极两类

![文本](/readmeimg/文本.jpg)

### 训练

batch_size=256 训练100轮

acc=0.9

# 代码结构

+---classroom			教室的监控视频
+---database			数据库相关操作
+---model
|   +---Classification		图像分类模型
|   |   +---weight			
|   +---Detection			物体检测模型
|   |   +---weight
|   +---Emotion
|   |   +---weight			情感分类模型
|   \---KeyWord			敏感词识别
+---resource			UI的图片资源文件
+---threads				线程类，用于实时监测
+---util				工具文件
\---view				qt界面文件
    +---css
    +---html
    +---images
    +---js
    +---ui

# 使用说明

GitHub地址：https://github.com/DragonistYJ/EduWatching.git

项目监控视频链接：链接：https://pan.baidu.com/s/1NBzpCEaPxgJ7TNQ8jr98qQ  提取码：1sru 

需要克隆代码到本地，首先按照requirements.txt中的依赖进行依赖安装

将ffmpeg加入到环境变量中

需要将教室的监控视频放到classroom文件夹下，按照如下目录格式命名

文件夹序号id从1开始连续增加，前置视频名称为{id}.mp4，后置视频名称为{id}_back.mp4

![QQ截图20191116165556](/readmeimg/QQ截图20191116165556.jpg)

运行Main.py脚本开始运行项目

# 功能介绍

## 实时监测线程

本项目中的所有监测功能均采用线程的方式进行识别

对于GPU的调用、UI的刷新都用线程锁进行控制

| 线程         | 功能                                                         |
| ------------ | ------------------------------------------------------------ |
| ClassBrief   | 每隔10秒对所有教室进行一次人数统计                           |
| StudentState | 每隔2秒首先监测教室内的人数，然后对于每一个学生进行状态分类  |
| Speech       | 每隔5秒对教师的语音进行语音识别，对识别出来的文字进行情感倾向的分类以及敏感词的识别 |
| Speech       | 每隔5秒对教师的语音进行情感的分类                            |
| Speech       | 每隔5秒对教师的语音进行语速的分类                            |
| TeacherVideo | 每隔2秒刷新教室后置摄像头的截图                              |

## 总监控界面

总监控页显示了对于所有教室的统筹监控，显示该课堂的基本信息以及教室内的学生人数

![main](/readmeimg/main.jpg)

用户可以在教学楼栏里按照校区、教学楼、楼层进行筛选

在状态栏里可以根据该教室是上课还是下课进行筛选

![main2](/readmeimg/main2.jpg)

在异常栏里只显示到课学生数量不足60%的教室

![main3](/readmeimg/main3.jpg)

## 实时监控界面

![realtime](/readmeimg/realtime.jpg)

在本页面中首先展示教室内学生状态的检测，对于玩手机的学生以用红色圈出，睡觉的学生用黄色圈出，站立的学生用蓝色圈出，同时描绘柱状图

下面展示教师的监控视频，检测教师的语速、情感，用折线图显示

右边显示识别出来的教师语音，有情感分析和敏感词识别

## 课程回看列表

在回看列表里面显示已经上过的课程的列表

该功能由于帐号管理问题，未开发完全

![Record](/readmeimg/Record.jpg)

## 课程回看

在这里可以看到对于一堂课中教师、学生出现的异常状态的记录

该功能由于帐号管理问题，未开发完全

![replay1](/readmeimg/replay1.jpg)

![replay2](/readmeimg/replay2.jpg)
