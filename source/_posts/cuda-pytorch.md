---
title: 安装CUDA以及Pytorch
date: 2018-03-18 21:05:04
tags:
---
最近在装实验室服务器的时候，遇到了一些问题，采坑无数，参考了很多博客，整合如下：

<!-- more -->

## 慎用 auto-remove：

事情的起因是想跑meta_crest的代码，很奇怪的是该程序只能使用python3，使用python2结果不对，因此需要安装python3版本的pytorch，pytorch只支持python3.5及3.6，而ubuntu 14.04的默认版本是python3.4，因此想先卸了python3.4，然后升级，一开始使用的命令是
```
sudo apt-get remove python3.4
```
执行完后发现输入python3后仍会启动python3.4，查了一些资料以后，比如这个[坑爹的连接](http://installion.co.uk/ubuntu/vivid/main/p/python3.4/uninstall/index.html)，然后敲下了这行命令：

```
sudo apt-get remove --auto-remove python3.4
```
弹出了一堆要卸载的包，大意是下列软件包是自动安装的并且现在不再被使用了，于是我觉得没问题就卸载了，然后悲剧的发现cuda和cudnn都没有了，装过的mxnet，pytorch，TensorFlow全部都跪了，只能浪费了一天时间重新装回来，后来查了一下，auto-remove还真是不能随便用，auto-remove的作用是卸载所有自动安装且不再使用的软件包，如[这篇博客](https://www.cnblogs.com/qiaoyanlin/p/6914236.html)所说：
> apt-get 提供了一个用于下载和安装软件包的简易命令行界面。
> 卸载软件包主要有这3个命令:
> - remove – 卸载软件包
> - autoremove – 卸载所有自动安装且不再使用的软件包
> - purge – 卸载并清除软件包的配置
>
> apt-get remove的行为我们很好理解，就是删除某个包的同时，删除依赖于它的包
> 例如： A 依赖于 B, B 依赖于 C
> apt-get remove 删除B的同时，将删除A(很好理解，A依赖于B，B被删了，A也就无法正常运行了)
>
> apt-get autoremove的行为重点是卸载所有自动安装
> 例如：C 依赖于 B, D 依赖于B, 且D没有被其他手动安装的包依赖
> apt-get remove C 将删除C, 同时提示你用apt-get autoremove去清除B,D apt-get autoremove C 将删除B, C, D aptitude remove C 将删除B, C, D
>
> 我的理解: 删除C, 那么B,D 这两个包既是自动安装的,且没有其他手动安装的包依赖于它们,
> 则可以判定B,D也是没必要的
>
> apt-get purge的行为卸载并清除软件包的配置，很容易理解
>
> 依赖性永远是个噩梦，不要考虑用 apt-get autoremove 卸载自己不熟悉的软件包
> apt-get remove卸载的是自己以及自己的下系
> apt-get autoremove则是卸载与自己相关而没有被其他手动安装包所依赖的包
> 如果你不是对一个包了解的话，有可能一用一下autoremove一下，你的系统就挂了。。。。。

另外还有两个惨痛的教训：
1. 最好通过包管理软件比如pip, apt-get等来安装卸载软件：

python3.6是通过源码编译的方式安装的，后来想卸载时惊奇的发现没有`make uninstall`这种命令，必须手动卸载，简直反人类，最后只能通过新建一个`tmp`文件夹，并通过`make`命令，将安装路径的`prefix`设置为该文件夹，看看装了什么东西，然后到`/usr/local`路径下（最早安装在这里）里面一个个去找，然后直接删除，这样会比较危险，也很麻烦，最好的方式还是用包管理软件，切记切记

2. python版本没有必要安装最新的，要跟系统的版本进行匹配：

一开始安装的是python3.6，安装pytorch的时候无法通过pip方式来安装对应版本的包，比如pyyaml，cycler等，是下载了源码以后编译安装的（也有可能是pip版本不对。。。），而安装完python3.5级对应版本的pip之后，缺什么就用pip3命令安装什么，简直不要太爽

## 安装CUDA-8.0

主要参考这篇博客：[Ubuntu 14.04安装CUDA-8.0](https://www.jianshu.com/p/35c7fde85968)

一定要先卸载驱动，然后再用`sh`来安装`.run`文件，另外询问是否安装`OpenGL`时选择`No`，要不然`CUDA-8.0`安装时会缺少一些库文件，比如`libcusolver`

### 0. FIRST OF ALL
#### 0.1 如果之前安装过，但失败了的同学，请敲下...
a).`.deb`安装失败的....
```
$ sudo apt-get --purge remove nvidia*
```
b)`.run`安装失败的....
执行
```
$ sudo /usr/local/cuda-8.0/bin/uninstall_cuda_8.0.pl
$ sudo /usr/bin/nvidia-uninstall
```
> 在 a) 或 b) 后,若仍安装有问题，请敲下

```
$ sudo apt-get autoremove --purge nvidia-*   #把nvidia驱动清个干干净净
$ sudo reboot
```

- !Note: sudo apt-get remove --purge nvidia-*这条指令并没卸载干净，可能存在驱动的冲突，导致安装不成功  

#### 0.2 建议来一本官方安装手册：
[NVIDIA CUDA INSTALLATION GUIDE FOR LINUX](http://docs.nvidia.com/cuda/cuda-installation-guide-linux/#axzz4RD7GVh1d)

### 1 PRE-INSTALLATION ACTION
#### 1.1 Verify you have a CUDA-Capable GPU
```
$ lspci | grep -i nvidia
```
我的机器显示：
```
01:00.0 3D controller: NVIDIA Corporation GF117M [GeForce 610M/710M/810M/820M / GT 620M/625M/630M/720M] (rev a1)
```
到[这里](https://developer.nvidia.com/cuda-gpus)验证型号

#### 1.2 Verify you have a Supported Version of Linux
```
$ uname -m && cat /etc/*release
```
结果显示：
```
x86_64
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
...
```
#### 1.3 Verify the System Has GCC Installed
```
$ gcc --version
```
结果显示：
```
gcc (Ubuntu 4.8.4-2ubuntu1~14.04.3) 4.8.4
...
```
#### 1.4 Verify the System has the Correct Kernel Headers and Development Packages Installed
查看正在运行的系统内核版本
```
$ uname -r
```
结果显示
```
4.4.0-45-generic
```
安装对应的kernels header和开发包：
```
$ sudo apt-get install linux-headers-$(uname -r)
```
#### 1.5 Download the NVIDIA CUDA Toolkit
到[这里](https://developer.nvidia.com/cuda-downloads)下载最新.run版本
或[这里](https://developer.nvidia.com/cuda-toolkit-archive)选择历史版本
![](/images/cuda.png)
下载完后，用MD5 检验，如果序号不和，得重新下载
```
$ md5sum cuda_8.0.27_linux.run
5639ffeb939ee58a81554d06bd084e15  cuda_8.0.27_linux.run
```
### 2. RUNFILE INSTALLATION
#### 2.1 Disabling Nouveau
```
$ lsmod | grep nouveau
```
如果有内容输出，则需禁掉`nouveau`
```
$ sudo vi /etc/modprobe.d/blacklist-nouveau.conf
```
添加如下内容：
```
blacklist nouveau
options nouveau modeset=0
```
保存退出(`:wq`)
执行
```
$ sudo update-initramfs –u
```
再执行
```
$ lsmod | grep nouveau
```
若无内容输出，则禁用成功
然后重启电脑
```
$ sudo reboot
```
#### 2.2 Reboot Into Text Mode
重启后，进入登录界面的时候，不要登录进入桌面(否则可能会失败，若不小心进入，请重启电脑)，直接按`Ctrl+Alt+F1`进入文本模式（命令行界面），登录账户。

关闭图形化界面
```
$ sudo service lightdm stop
```
切换到`cuda_8.0.27_linux.run`的目录，执行
```
$ sudo sh cuda_8.0.27_linux.run
```
> !Note:安装的时候，要让你先看一堆文字（EULA)，我们直接不停的按空格键到100%;
> 遇到提示是否安装openGL ，选择no,其他的可以一路accept, yes或回车

安装成功后，会显示**installed**，否则会显示**failed**。

重启图形化界面
```
$ sudo service lightdm start
```
登录时能进入桌面，不会一直在重复登录，成功已近大半。

> !Note:如果出现重复登陆情况，请卸载cuda,然后重装。
> 原因：是OpenGL与NVIDIA发生了什么什么的
> 卸载：由于登陆进入不到图形用户界面（GUI），但我们可以进入到文本用户界面(TUI)(TUI很酷有没有?)


- 在登陆界面时，按Ctrl + Alt + f1,进入TUI
- 执行
```
$ sudo /usr/local/cuda-8.0/bin/uninstall_cuda_8.0.pl
$ sudo /usr/bin/nvidia-uninstall
```
- 然后重启
```
$ sudo reboot
```
- 重新安装.run(安装时请留眼，在提示是否安装OpenGL时，应该选no)  

#### 2.3 Device Node Verification
执行
```
$ ls /dev/nvidia*
```
可能出现a), b), c)，d)三种结果，请对号入座。前方高能！

a) 若结果显示
```
/dev/nvidia0  /dev/nvidiactl  /dev/nvidia-uvm
```
或显示出类似的信息，应该有三个（包含一个类似/dev/nvidia-nvm的），则安装成功

b) 如果运气有点背，结果是这样的
```
ls: cannot access /dev/nvidia*: No such file or directory
```
或是这样的，只出现
```
/dev/nvidia0  /dev/nvidiactl
```
中的一个或两个，但没有/dev/nvidia-num

莫方，也许还有希望（我在安装时就是这种情况。。。）按照官方的做法：

把下面的`.sh`文件随便命个名(我命名为`Nka.sh`)
```
#!/bin/bash

/sbin/modprobe nvidia

if [ "$?" -eq 0 ]; then
  # Count the number of NVIDIA controllers found.
  NVDEVS=`lspci | grep -i NVIDIA`
  N3D=`echo "$NVDEVS" | grep "3D controller" | wc -l`
  NVGA=`echo "$NVDEVS" | grep "VGA compatible controller" | wc -l`

  N=`expr $N3D + $NVGA - 1`
  for i in `seq 0 $N`; do
    mknod -m 666 /dev/nvidia$i c 195 $i
  done

  mknod -m 666 /dev/nvidiactl c 195 255

else
  exit 1
fi

/sbin/modprobe nvidia-uvm

if [ "$?" -eq 0 ]; then
  # Find out the major device number used by the nvidia-uvm driver
  D=`grep nvidia-uvm /proc/devices | awk '{print $1}'`

  mknod -m 666 /dev/nvidia-uvm c $D 0
else
  exit 1
fi
```
然后执行
```
$ sudo chmod +x Nka.sh
$ sudo ./Nka.sh
$  ls /dev/nvidia*
/dev/nvidia0  /dev/nvidiactl  /dev/nvidia-uvm
```
成功！

> 1， 这种做不太友好，我的意思是，当下次重启电脑时，你使用ls /dev/nvidia*指令时，你是看不到那三个nvidia的文件了。所以你又得手动执行
sudo ./Nka.sh指令了，是不是很烦！其实上面的.sh文件是startup scipt，也就是启动脚本。顾名思义，就是在系统启动时，自动加载的。哈，这么棒的功能就是我们想要的。
2， 添加启动脚本的方法大致有两种，我就此介绍一种最傻瓜化的方法。  

执行
```
$ sudo vi /etc/rc.local
```
如果你是第一次打开这个文件，它应该是空的(除了一行又一行的#注释项外)。这文件的第一行是
```
#!/bin/sh -e
```
把-e去掉（这步很重要，否则它不会加载这文本的内容）
然后把`Nka.sh`的内容除了`#!/bin/bash`外复制到其中，(before **exit 0**)保存退出。
下次重启时，你应该能直接看到`/dev`目录下的三个`nvidia`的文件
```
$ ls /dev/nvidia*
/dev/nvidia0 /dev/nvidiactl /dev/nvidia-uvm
```
c) 如果人品实在不好（我就遇过几次。。。），结果是这样的
```
modprobe: ERROR: could not insert 'nvidia_uvm': Operation not permitted
```
少年，我救不了你了。但是**winney**大神可以。（在此谢过她了，阿 里 嘎 多！）

当出现这种情况时，可能是驱动打起架来了。
执行
```
$ sudo apt-get autoremove --purge nvidia-* #把nvidia驱动清个干干净净
$ sudo reboot         #一定记得重启，不然你会后悔的!
```
然后
```
$ sudo ./Nka.sh
$ ls /dev/nvidia*
```
这时，应该可以见到
```
/dev/nvidia0 /dev/nvidiactl /dev/nvidia-uvm
```
d) 未知，有点悲伤的告诉你，少年，我只能帮到这了,建议网上另寻方案，或重装.run。Gook Luck!
### 3 POST-INSTALLATION ACTIONS
#### 3.1 Environment Setup
打开系统配置文件
```
$ sudo vi /etc/profile
```
在文件最后添加
```
export PATH=/usr/local/cuda-8.0/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-8.0/lib64:$LD_LIBRARY_PATH
```
保存退出

执行
```
$ source /etc/profile
```
让文件立即生效

至此cuda 8.0安装完毕。

### 3.2 Verify the Installation
#### 3.2.1 Verify the Driver Version
敲入
```
$ cat /proc/driver/nvidia/version
```
结果显示
```
NVRM version: NVIDIA UNIX x86_64 Kernel Module  361.77  Sun Jul 17 21:18:18 PDT 2016
GCC version:  gcc version 4.8.4 (Ubuntu 4.8.4-2ubuntu1~14.04.3)
```
或之类的东东

#### 3.2.2 Verify CUDA Toolkit
敲入
```
$ nvcc -V
```
结果显示
```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2016 NVIDIA Corporation
Built on Wed_May__4_21:01:56_CDT_2016
Cuda compilation tools, release 8.0, V8.0.26
```
> !Note: 如果是这样的：
```
The program 'nvcc' is currently not installed. You can install it by typing:
sudo apt-get install nvidia-cuda-toolkit
```
莫方，确认下`/etc/profile`的配置环境是否正确

>即使什么都没改，可能忘了这一步,或是之前执行了，但过了有段时间，且又还没重启电脑。因为source /etc/profile是临时生效，重启电脑才是永久生效

执行
```
$ source /etc/profile
```
再执行(应该就有显示了）
```
$ nvcc -V
```
#### 3.2.3 Complie sample
`cd`进NVIDIA_CUDA-8.0_Samples目录
执行
```
$  make
```
> !Note: 这区间大概需要十几到二十分钟，请耐心等待。建议来杯caffe

运行完后，编译结果会放在`NVIDIA_CUDA-8.0_Samples`目录下的`bin`目录

#### 3.2.3 Running the Binaries
`cd`进`bin`目录里面的里面的里面，知道看到一堆可执行文件（菱形的图标），大概是`~/NVIDIA_CUDA-8.0_Samples/bin/x86_64/linux/release`

执行
```
$ ./deviceQuery
```
结果显示
```
./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "GeForce GT 720M"
  CUDA Driver Version / Runtime Version          8.0 / 8.0
  CUDA Capability Major/Minor version number:    2.1
  Total amount of global memory:                 1985 MBytes (2081226752 bytes)
  ( 2) Multiprocessors, ( 48) CUDA Cores/MP:     96 CUDA Cores
  GPU Max Clock rate:                            1250 MHz (1.25 GHz)
  Memory Clock rate:                             800 Mhz
  Memory Bus Width:                              64-bit
  L2 Cache Size:                                 131072 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(65536), 2D=(65536, 65535), 3D=(2048, 2048, 2048)
  Maximum Layered 1D Texture Size, (num) layers  1D=(16384), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(16384, 16384), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 32768
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  1536
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (65535, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 1 copy engine(s)
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            No
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 1 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 8.0, CUDA Runtime Version = 8.0, NumDevs = 1, Device0 = GeForce GT 720M
Result = PASS
```
或之类的东东，且最后是`Result = PASS`,若失败`Result = FAIL`

再来一个，执行
```
$ ./bandwidthTest
```
结果显示
```
[CUDA Bandwidth Test] - Starting...
Running on...

 Device 0: GeForce GT 720M
 Quick Mode

 Host to Device Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)    Bandwidth(MB/s)
   33554432         3220.9

 Device to Host Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)    Bandwidth(MB/s)
   33554432         3271.9

 Device to Device Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)    Bandwidth(MB/s)
   33554432         9772.8

Result = PASS

NOTE: The CUDA Samples are not meant for performance measurements. Results may vary when GPU Boost is enabled.
```
少年，恭喜你！你已成功安装`cuda-8.0`,接下来就可以愉快的玩耍了

## 安装python3.5

ubuntu14.04版本无法直接通过apt-get来安装python3.5，需要先加入第三方维护的ppa,update后才能安装，执行如下所示命令：

```
sudo add-apt-repository ppa:fkrull/deadsnakes
sudo apt-get update
sudo apt-get install python3.5
```
安装后，输入python启动的是python2.7.2, 由于Ubuntu底层采用的是Python2.*，Python3.*与Python2.*是不兼容的，但是不能卸载Python2,随意卸载会出现意想不到的后果,最好不要随便更改python的指向，而是应该通过python3的方式来启动python3.5，但是安装完后输入python3启动的并不是python3.5而是系统默认的python3.4，可以采用以下方式来更改python3的指向：

```
1. 新建 ~/.bash_aliases
2. 输入 alias python3=python3.5
3. 更新 source ~/.bash_aliases
```
此后输入python3,就会启动python3.5

## 安装pip3.5:

系统自带的pip3是针对系统安装的python3.5的，因此需要安装python3.5版本对应的pip3：

```
wget https://bootstrap.pypa.io/get-pip.py
sudo python3 get-pip.py
sudo pip3 install setuptools --upgrade
```
安装完毕后，同样在`~/.bash_aliases`中加入`alias pip3=pip3.5`来进行替换。

## 安装tensorflow:

由于机器安装的是CUDA-8.0，而TensorFlow仅在1.2版本以下支持CUDA-8.0，因此只能安装1.2版本，直接采用pip方式进行安装

```
sudo pip install tensorflow-gpu==1.2.0
```
