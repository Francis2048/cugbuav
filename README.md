### 

# ROS下基于视觉的无人机室内定点飞行全程记录（T265+PX4+mavros）

​	

[TOC]

本文记录利用Pixhawk2.4.8、[Ubuntu](https://so.csdn.net/so/search?q=Ubuntu&spm=1001.2101.3001.7020) 20.04/18.04、ROS Notetic、T265、机载电脑实现四旋翼无人机在室内无GPS情况下的定点稳定飞行

本文作者：李绍华 中国地质大学北京 

欢迎各位交流学习！

qq:798750386

## 一、机载环境（20.04）建议18.04 ，由于网上没有20.04，故出此篇

### 1.烧录系统

硬件：Raspberry Pi 4B(4g或8g版本)

系统：Ubuntu 20.04 server LTS

1. 从官网上下载ubuntu20.04LTS镜像，用官网image软件进行烧录

2. 打开树莓派，初次登陆账号密码均为ubuntu，首次登陆系统会强制要求更换密码，记得第一次询问的是原密码，即ubuntu

3. 用读卡器读sd卡，在config文件里面添加以下内容（7寸显示器）

   ```
   disable_overscan=1
   hdmi_force_hotplug=1  # 强制树莓派使用HDMI端口，即使树莓派没有检测到显示器连接仍然使用HDMI端口。
   #该值为0时允许树莓派尝试检测显示器，当该值为1时，强制树莓派使用HDMI。
   hdmi_drive=2  # 可以使用该配置项来改变HDMI端口的电压输出：
   #1-DVI输出电压。该模式下，HDMI输出中不包含音频信号。
   #2-HDMI输出电压。该模式下，HDMI输出中包含音频信号。
   hdmi_group=2  # 决定的分辨率
   #DMT分辨率是hdmi_group=2，计算机显示器使用的分辨率；hdmi_group=1是CEA分辨率 ，CEA规定的电视规格分辨率
   hdmi_mode=18
   ```

4. 在network文件里面，添加wifi信息，每行缩进2空格

   ```
   wifis:
     wlan0:
       dhcp4: true
       optional: true
       access-points:
        "wifi名称":
           password: "密码"
   ```

5. 换apt源，教程在设置网络源中有讲

6. 随后`sudo apt update `以及` sudo apt upgrade `更新apt源

7. 安装中文环境

   ```
   sudo apt install language-pack-zh-hans language-pack-zh-hans-base language-pack-gnome-zh-hans language-pack-gnome-zh-hans-base
   sudo apt install `check-language-support -l zh`
   sudo reboot #重启
   ```


**进行root密码的设置**

`sudo passwd root`

**完成后设置后的系统:**

用户名:ubuntu 密码 uav2022cugb

管理员:root 密码 1

**换源：**

https://mirror.tuna.tsinghua.edu.cn/help/ubuntu-ports/

### 2. 安装桌面以及远程桌面

#### 2.1安装和配置ssh

```
sudo apt install openssh-server
sudo apt install openssh-client
配置ssh_config：
sudo vi /etc/ssh/ssh_config
将PasswordAuthentication设置为yes,之后重启ssh:
sudo /etc/init.d/ssh restart
```

安装 Gnome(桌面环境)

```
sudo apt update

sudo apt-get upgrade

sudo apt install ubuntu-desktop
```

#### 2.2vino实现桌面共享

将ubuntu系统的桌面共享给其它用户，通过vnc viewer 5900端口，实现对ubuntu桌面的控制和访问，类似云桌面。

在ubuntu系统上，输入以下命令，安装配置vino。

```
sudo apt-get install vino
```

打开ubuntu系统设置->共享->屏幕共享，按下图进行设置。密码最多可以设置8位，网络根据自己实际情况选择。

![image-20220813125441030](%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B.assets/image-20220813125441030.png)

`gsettings set org.gnome.Vino require-encryption false`关闭加密策略

在windows上下载vnc-viewer软件，下载链接：[点击链接到下载页面](https://www.realvnc.com/en/connect/download/viewer/)。下载安装完成后，打开vnc-viewer。

在VNC Server一栏输入ubuntu系统的ip地址和端口，vino默认使用端口5900，例如：192.168.2.176。

![image-20220813125636625](%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B.assets/image-20220813125636625.png)

点击生成的共享桌面，在弹出的页面输入ubuntu系统登陆密码。输出完成后点击OK，就会弹出远程分享的ubuntu桌面了。

提供的镜像中vnc密码为**uav**

#### 2.3显卡欺骗器替代方案

Ubuntu默认的xserver不能工作在无显示器环境下，需要创建虚拟显示器，推荐安装xserver-xorg-video-dummy

**安装命令：**
`sudo apt-get install xserver-xorg-video-dummy`
安装结束后修改配置文件xorg.conf（如果没有创建一个），保存重启即可使用远程桌面。

`sudo nano /usr/share/X11/xorg.conf.d/xorg.conf`

```
Section "Device"
    Identifier  "Configured Video Device"
    Driver      "dummy"
EndSection

Section "Monitor"
    Identifier  "Configured Monitor"    
    HorizSync 31.5-48.5
    VertRefresh 50-70
EndSection

Section "Screen"
    Identifier "Default Screen"
    Monitor "Configured Monitor"
    Device "Configured Video Device"
    DefaultDepth 24
    SubSection "Display"
    Depth 24
    Modes "1024x800"
    EndSubSection
EndSection
```

### 3.安装ros

```
wget http://fishros.com/install -O fishros && . fishros
```

这里使用鱼香ros一键安装ros的方法，真香

### 4.安装mavros

用于`apt-get`安装：

`sudo apt-get install ros-kinetic-mavros ros-kinetic-mavros-extras`

然后安装[GeographicLib （打开新窗口）](https://geographiclib.sourceforge.io/)通过运行`install_geographiclib_datasets.sh`脚本获取数据集：

```
wget https://raw.githubusercontent.com/mavlink/mavros/master/mavros/scripts/install_geographiclib_datasets.sh
sudo ./install_geographiclib_datasets.sh
```

**报错1：**

运行：`./install_geographiclib_datasets.`

`-bash: ./install_geographiclib_datasets.sh: Permission denied`

解决方法：

`sudo chmod +x ./install_geographiclib_datasets.sh`

`sudo ./install_geographiclib_datasets.sh`

**报错2**：

`Connecting to raw.githubusercontent.com failed: Connection refused`

```bash
sudo vim /etc/hosts
#绑定host
185.199.108.133 raw.githubusercontent.com
```

如果无效则在[https://www.ipaddress.com/](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.ipaddress.com%2F) 查询raw.githubuercontent.com的真实IP。

经过漫长的等待Terminial中出现如下内容：

```
Installing GeographicLib geoids egm96-5
Installing GeographicLib gravity egm96
Installing GeographicLib magnetic emm2015
```

可以再运行一次`sudo bash ./install_geographiclib_datasets.sh   `

```
GeographicLib geoids dataset egm96-5 already exists, skipping
GeographicLib gravity dataset egm96 already exists, skipping
GeographicLib magnetic dataset emm2015 already exists, skipping
```

可以确保mavros已经成功安装

### 5.配置T265环境

本文只涉及部署Intel的T265相机。分为两个部分，SDK的安装和ROS节点的安装。前提是ROS系统已经被部署在树莓派上。

#### 0. 预操作：扩大Swap分区

Ubuntu 默认的Swap分区太小，编译时会直接卡死，并且不会报错，所以首先要扩大swap分区。另外，编译操作会让树莓派温度升高的很厉害，需要安装好散热风扇。 查看当前的交换空间大小，我的默认是100M.

```shell
free -m 
```

##### **1.建立交换空间文件**

```shell
cd /opt/
sudo mkdir swap_temp #名字任意起
cd swap_temp
sudo touch swap
```

##### **2.设置交换文件的大小**

```shell
sudo dd if=/dev/zero of=/opt/swap_temp/swap bs=1024 count=3048000  # 我这里设置的是3G
```

等一会儿，结束时才会有回显，我的写入速度是20M每秒，3G需要2分钟左右 结束后返回：

```text
3048000+0 records in
3048000+0 records out
2097152000 bytes (2.9 GB, 3.0 GiB) copied, 242.095 s, 18.7 MB/s
```

##### **3.设置为交换空间**

```shell
sudo mkswap /opt/swap_temp/swap
```

##### **4.启用交换空间**

```shell
sudo swapon /opt/swap_temp/swap
```

现在已经可以使用了，通过free -m查看

##### **5.写入分区**

```shell
sudo vim /etc/fstab
文件最后加入
/opt/swap_temp/sawp /swap swap defaults 0 0
```

#### 1.Intel Realsense SDK的安装

##### **1.安装依赖包**

```shell
sudo apt-get update && sudo apt-get upgrade && sudo apt-get dist-upgrade
sudo apt-get install git cmake libssl-dev libusb-1.0-0-dev pkg-config libgtk-3-dev
sudo apt-get install libglfw3-dev libgl1-mesa-dev libglu1-mesa-dev
```

第二行的是核心依赖，必装。第三行是3D相关的依赖，如果不打算使用realsense-viewer，可以不装，树莓派性能有限，安装SDK的目的是为了安装好驱动，不需要执行太多本机操作，建议内存卡小的不用装了。

##### **3.下载Realsense SDK**

```shell
git clone https://github.com/IntelRealSense/librealsense.git
```

##### **4.编译准备**

```shell
cd librealsense
mkdir build && cd build
cmake ../ -DCMAKE_BUILD_TYPE=Release -DBUILD_EXAMPLES=true \
-DFORCE_RSUSB_BACKEND=ON -DBUILD_WITH_TM2=false -DIMPORT_DEPTH_CAM_FW=false
```

**-DFORCE_RSUSB_BACKEND=ON 必选，强制LIBUVC后端，否则你要自己给内核打补丁。**

##### 5.编译

```sudo make uninstall && make clean && make -j2 && sudo make install -2
我自己的4B需要编译三小时左右。中间交换空间最大占用为1.6G。

##### **6.设置udev规则**

​```shell
sudo ./scripts/setup_udev_rules.sh
```

主要是为了识别设备，最重要的也就是这里

#####  **7.测试**

```text
realsense-viewer
```

我的机载电脑没有开图形界面，一般都是VNC连接上去。

#### 2. 编译ROS驱动

目的是为了让ROS节点可以订阅T265发回的IMU信息。 英特尔官方发布了两种安装方式，1是通过apt的方式安装二进制文件，2是通过源码编译。但是前提是系统里已经安装好了对应的ROS系统，我的是18.04+Melodic

##### 2.1 APT-Get 安装

```shell
# 在 Ubuntu 16.04 上安装ROS Kinetic，在Ubuntu 18.04上安装ROS Melodic或在 Ubuntu 20.04 上安装 ROS Noetic 。
export ROS_VER=noetic
```

安装

```shell
sudo apt-get install ros-$ROS_VER-realsense2-camera
```

用于3D显示的库

```text
sudo apt-get install ros-$ROS_VER-realsense2-description
```

注意：

- 这种方法安装的librealsense2总是落后于最新发布的版本

##### 2.2 源码编译

1.创建工作目录

工作目录的名字不一定要是catkin_ws

```shell
mkdir -p ~/realsense_ws/src
cd ~/catkin_ws/src/
```

2.克隆源码

```shell
git clone https://github.com/IntelRealSense/realsense-ros.git
cd realsense-ros/
git checkout `git tag | sort -V | grep -P "^\d+\.\d+\.\d+" | tail -1`
cd ..
```

3.编译

```shell
catkin_init_workspace
cd ..
catkin_make clean
catkin_make -DCATKIN_ENABLE_TESTING=False -DCMAKE_BUILD_TYPE=Release
catkin_make install
echo "source ~/catkin_ws/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

##### 3.使用

启动节点

```shell
roslaunch realsense2_camera rs_camera.launch
```



```C++
//1、安装安装依赖项
sudo apt-get install libudev-dev pkg-config libgtk-3-dev
sudo apt-get install libusb-1.0-0-dev pkg-config
sudo apt-get install libglfw3-dev
sudo apt-get install libssl-dev
sudo apt-get install libglu1-mesa-dev
//2、下载安装包
sudo git clone https://github.com/IntelRealSense/librealsense
或者下载指定版本
sudo git clone https://github.com/IntelRealSense/librealsense/releases/tag/v2.16.1
//3、编译安装
cd librealsense
mkdir build
cd build
cmake ../ -DBUILD_EXAMPLES=true
make
sudo make install
//4、测试
realsense-viewer 
```



```C++
//注册服务器公钥
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE || sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-key F6E65AC044F831AC80A06380C8B3A55A6F3EFCDE

//将服务器添加到公钥之中
sudo add-apt-repository "deb https://librealsense.intel.com/Debian/apt-repo $(lsb_release -cs) main" -u

//更新下载源
sudo apt-get update

//安装库文件
sudo apt-get install librealsense2-dkms
sudo apt-get install librealsense2-utils
//安装开发人员包和调试工具包
sudo apt-get install librealsense2-dev
sudo apt-get install librealsense2-dbg
    
//补充配置相关驱动和依赖
sudo apt-get install ros-melodic-realsense2-camera
sudo apt install ros-melodic-cv-bridge ros-melodic-image-transport ros-melodic-tf ros-melodic-diagnostic-updater ros-melodic-ddynamic-reconfigure
//安装Realsense SDK
git clone https://github.com/IntelRealSense/librealsense.git
//编译准备工作
cd librealsense
mkdir build
cd build
cmake ../ -DCMAKE_BUILD_TYPE=Release -DBUILD_EXAMPLES=true \
-DFORCE_RSUSB_BACKEND=ON -DBUILD_WITH_TM2=false -DIMPORT_DEPTH_CAM_FW=false
//编译
sudo make uninstall && make clean && make  && sudo make install
//配置ROS Wrapper for Intel RealSense
//创建ROS工作空间
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/src
//下载realsense-ros 2.2.7版本，安装依赖项
git clone -b 2.2.7 https://github.com/IntelRealSense/realsense-ros.git
sudo apt-get install ros-melodic-ddynamic-reconfigure
//初始化工作空间
catkin_init_workspace
cd ..
//编译工作空间
catkin_make clean
catkin_make -DCATKIN_ENABLE_TESTING=False -DCMAKE_BUILD_TYPE=Release
catkin_make install
//增加环境变量
gedit ~/.bashrc
source ~/catkin_ws/devel/setup.bash
export ROS_PACKAGE_PATH=${ROS_PACKAGE_PATH}:~/catkin_ws/
```

### 6.配置坐标转换包

```C++
//下载坐标转换工具，编译工作空间
mkdir -p ~/vision_ws/src
cd vision_ws/src
git clone https://github.com/thien94/vision_to_mavros.git
catkin_init_workspace
cd ..
catkin_make
//配置环境变量
gedit ~/.bashrc
source ~/vision_ws/devel/setup.bash
export ROS_PACKAGE_PATH=${ROS_PACKAGE_PATH}:~/vision_ws/
//修改配置参数
cd vision_ws/src/vision_to_mavros/launch
vim t265_all_nodes.launch
//将<include file="$(find mavros)/launch/apm.launch">修改为<include file="$(find mavros)/launch/px4.launch">
```

##  二、仿真环境配置（18.04）


仿真环境不可以运行在树莓派机载电脑上，我们可以采取双系统或者虚拟机进行安装ubuntu18.04的仿真环境

双系统教程：

`https://www.jianshu.com/p/fe4e3915495e`

虚拟机教程:

`https://blog.csdn.net/WangZijun_1996/article/details/80163507`

***划重点：请一定按照先改系统参数，设置网络源→搭建px4工具链→配置ros环境→安装mavros→配置T265→安装QGC的顺序进行，否则会出现px4在环仿真不通过等报错情况***

### 1.设置网络源

`sudo gedit /etc/apt/sources.list`

家庭网设置ailiyun源

```
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse    
```

教育网设置ustc源

```
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
```

更换完成后进行更新

`sudo apt-get update`

`sudo apt-get upgrade`

### 2.搭建px4工具链

```
git clone https://github.com/PX4/PX4-Autopilot.git
mv PX4-Autopilot PX4_Firmware
cd PX4_Firmware
git checkout -b xtdrone/dev v1.11.0-beta1
git submodule update --init --recursive
make px4_sitl_default gazebo
```

编译完成后，会弹出Gazebo界面，将其关闭即可。

修改 ~/.bashrc，加入以下代码,注意路径匹配，前两个source顺序不能颠倒。

```
source ~/catkin_ws/devel/setup.bash
source ~/PX4_Firmware/Tools/setup_gazebo.bash ~/PX4_Firmware/ ~/PX4_Firmware/build/px4_sitl_default
export ROS_PACKAGE_PATH=$ROS_PACKAGE_PATH:~/PX4_Firmware
export ROS_PACKAGE_PATH=$ROS_PACKAGE_PATH:~/PX4_Firmware/Tools/sitl_gazebo
```

再运行：

```
source ~/.bashrc
```

然后运行如下命令,此时会启动Gazebo.

```
cd ~/PX4_Firmware
roslaunch px4 mavros_posix_sitl.launch
```

并运行

```
rostopic echo /mavros/state
```

若connected: True,则说明MAVROS与SITL通信成功。如果是false，一般是因为.bashrc里的路径写的不对，请仔细检查。

```
---
header:
seq: 11
stamp:
secs: 1827
nsecs: 173000000
frame_id: ''
connected: True
armed: False
guided: False
manual_input: True
mode: "MANUAL"
system_status: 3
---
```

然后需要安装地面站QGroundControl，点此[安装链接](https://docs.qgroundcontrol.com/en/getting_started/download_and_install.html)。启动后,将出现下图 所示画面。注意Ubuntu16.04没法直接使用QGroundcontrol 版本4系列（可以使用版本3系列），Ubuntu16.04需要源码编译版本4系列，请仔细查看[安装链接](https://docs.qgroundcontrol.com/en/getting_started/download_and_install.html)。

### 3.配置ROS环境

```C++
//此次使用的是中科大源，若阅读者所用非此源，可参考http://wiki.ros.org/ROS/Installation/UbuntuMirrors更换相关命令
sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.ustc.edu.cn/ros/ubuntu/ `lsb_release -cs` main" > /etc/apt/sources.list.d/ros-latest.list'
sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
sudo apt update
sudo apt install ros-melodic-desktop-full
sudo apt install python-rosdep
sudo rosdep init
rosdep update
echo "source /opt/ros/melodic/setup.bash" >> ~/.bashrc
source ~/.bashrc
sudo apt install python-rosdep python-rosinstall python-rosinstall-generator python-wstool build-essential
```

### 4.安装mavros

```
sudo apt install ros-melodic-mavros ros-melodic-mavros-extras 		# for ros-melodic 
wget https://gitee.com/robin_shaun/XTDrone/raw/master/sitl_config/mavros/install_geographiclib_datasets.sh
sudo chmod a+x ./install_geographiclib_datasets.sh
sudo ./install_geographiclib_datasets.sh #这步需要装一段时间,请耐心等待PX4配置
```

### 5.安装gazebo9.19

1.卸载gazebo9.0

```sudo apt-get remove gazebo9*
sudo apt-get remove gazebo9*
```

2.安装gazebo9.19

```
sudo sh -c 'echo "deb http://packages.osrfoundation.org/gazebo/ubuntu-stable `lsb_release -cs` main" > /etc/apt/sources.list.d/gazebo-stable.list'
cat /etc/apt/sources.list.d/gazebo-stable.list
#如果出现deb http://packages.osrfoundation.org/gazebo/ubuntu-stable xenial main表示没问题
wget https://packages.osrfoundation.org/gazebo.key -O - | sudo apt-key add -
sudo apt-get update
sudo apt-get install gazebo9=9.1*
```

3.安装过后输入gazebo,如果不能弹出gazebo窗口，且报错类似

```
error: gzserver: symbol lookup error: /usr/lib/x86_64-linux-gnu/libsdformat.so.4: undefined symbol: _ZN8ignition4math15SemanticVersionC1ERKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
```

执行`sudo apt upgrade libignition-math2`

4.安装相关依赖项

```
sudo apt-get install gazebo9*
sudo apt-get install libgazebo9-dev
sudo apt-get install ros-melodic-gazebo-ros
sudo apt-get install ros-melodic-gazebo-ros-control
sudo apt-get install ros-melodic-gazebo-ros-pkgs
```

### 6.设置第一个工作空间并配置官方例程

由于mavros采用二进制安装，所以只需要配置CMakelist文件即可

```C++
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws
catkin build
source devel/setup.bash
cd catkin_ws/src
```

1) 在刚刚建立的catkin_ws/src目录下建立一个offboard包

该包依赖于roscpp mavros geometry_msgs

`catkin_create_pkg offboard roscpp mavros geometry_msgs`

2) 进入offboard/src下新建一个.cpp文件，把官网下的代码复制过去：

cd offboard/src
gedit offboard_node.cpp
3) 回到offboard目录下修改CMakeList.txt文件
主要修改有3点：

在depends处取消注释，如下图

![在这里插入图片描述](%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B.assets/20210309165707120.png)

添加可执行文件 offboard_node.cpp，如下图

![img](%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B.assets/20210309165841464.png)

添加库链接，如下图：

![在这里插入图片描述](%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B.assets/20210309165923767-1660992590808.png)

4）回到根目录下，build

```
新开一个终端
cd catkin_ws
catkin build
source devel/setup.bash
```

5）可以运行示例了
首先，运行gazebo仿真

```
打开一个新终端
cd PX4-Autopilot
make px4_sitl_default gazebo
```

然后运行MAVROS,链接到本地ROS

```
开一个新终端
roslaunch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557"
```

最后运行offboard示例

```
开一个新终端
rosrun offboard offboard_node
```

## 三、无人机offboard控制仿真

使用PX4+mavros+gazebo实现无人机offboard控制仿真
首先需要安装好ROS，gazebo，PX4，mavros，QGroundControl等工具，安装过程参考链接：

操作系统：Ubuntu18.04

ROS：http://wiki.ros.org/cn/melodic/Installation/Ubuntu（官网教程链接）

 安装其中桌面完整版（ros-melodic-desktop-full）自带gazebo

PX4：https://www.cnblogs.com/cporoske/p/11630426.html

mavros：https://www.cnblogs.com/cporoske/p/11641477.html

 其中执行 ./install_geographiclib_datasets.sh 步骤时若时间过长参考解决方法：

 https://gitee.com/MrZhaosx/geographic-lib/tree/master

QGC：https://docs.qgroundcontrol.com/master/en/getting_started/download_and_install.html （官网链接）

实现offboard模式速度控制参考链接：
https://blog.csdn.net/wbzhang233/article/details/106727276/

https://www.youtube.com/watch?v=2jksI-S3ojY

（未通过程序实现，通过命令行实现）

**实现offboard模式sin路线飞行控制：**

1. 找到PX4的安装路径，打开gazebo仿真并启动PX4：

   ```
   cd Firmware
   make px4_sitl gazebo_iris
   ```

2. 打开QGC地面站

3. 建立mavros与PX4的连接

   `roslaunch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557"`

4. 找到ROS工作空间，创建功能包

   `catkin_create_pkg offboard_sin roscpp std_msgs geometry_msgs mavros_msgs`

5. 在功能包的src路径下创建offboard_sin_node.cpp文件写入如下代码

   ```C++
   /**
    * @file offb_node.cpp
    * @brief Offboard control example node, written with MAVROS version 0.19.x, PX4 Pro Flight
    * Stack and tested in Gazebo SITL
    */
   
   #include <ros/ros.h>
   #include <geometry_msgs/PoseStamped.h>
   #include <geometry_msgs/Vector3.h>
   #include <mavros_msgs/CommandBool.h>
   #include <mavros_msgs/SetMode.h>
   #include <mavros_msgs/State.h>
   
   #define PI acos(-1)
   
   mavros_msgs::State current_state;
   geometry_msgs::PoseStamped current_position;
   
   void state_cb(const mavros_msgs::State::ConstPtr& msg){
       current_state = *msg;
   }
   void getpointfdb(const geometry_msgs::PoseStamped::ConstPtr& msg){
       ROS_INFO("x: [%f]", msg->pose.position.x);
       ROS_INFO("y: [%f]", msg->pose.position.y);
       ROS_INFO("z: [%f]", msg->pose.position.z);
       current_position = *msg;
   }
   
   int main(int argc, char **argv)
   {
       ros::init(argc, argv, "offb_node");
       ros::NodeHandle nh;
   
       ros::Subscriber state_sub = nh.subscribe<mavros_msgs::State>
               ("mavros/state", 10, state_cb);
               
       ros::Subscriber get_point = nh.subscribe<geometry_msgs::PoseStamped>
               ("mavros/local_position/pose", 10, getpointfdb);
               
       ros::Publisher local_pos_pub = nh.advertise<geometry_msgs::PoseStamped>
               ("mavros/setpoint_position/local", 10);
       ros::ServiceClient arming_client = nh.serviceClient<mavros_msgs::CommandBool>
               ("mavros/cmd/arming");
       ros::ServiceClient set_mode_client = nh.serviceClient<mavros_msgs::SetMode>
               ("mavros/set_mode");
   
       //the setpoint publishing rate MUST be faster than 2Hz
       ros::Rate rate(20.0f);
   
       // wait for FCU connection
       while(ros::ok() && !current_state.connected){
           ros::spinOnce();
           rate.sleep();
       }
   
       geometry_msgs::PoseStamped pose;
       pose.pose.position.x = 0;
       pose.pose.position.y = 0;
       pose.pose.position.z = 3;
       
       
       
   
       //send a few setpoints before starting
       for(int i = 100; ros::ok() && i > 0; --i){
           local_pos_pub.publish(pose);
           ros::spinOnce();
           rate.sleep();
       }
   
       mavros_msgs::SetMode offb_set_mode;
       offb_set_mode.request.custom_mode = "OFFBOARD";
   
       mavros_msgs::CommandBool arm_cmd;
       arm_cmd.request.value = true;
   
       ros::Time last_request = ros::Time::now();
   
       while(ros::ok()){
           if( current_state.mode != "OFFBOARD" &&
               (ros::Time::now() - last_request > ros::Duration(5.0f))){
               if( set_mode_client.call(offb_set_mode) &&
                   offb_set_mode.response.mode_sent){
                   ROS_INFO("Offboard enabled");
               }
               last_request = ros::Time::now();
           } else {
               if( !current_state.armed &&
                   (ros::Time::now() - last_request > ros::Duration(5.0f))){
                   if( arming_client.call(arm_cmd) &&
                       arm_cmd.response.success){
                       ROS_INFO("Vehicle armed");
                   }
                   last_request = ros::Time::now();
               }
           }
           
           if((abs(current_position.pose.position.x-pose.pose.position.x)<0.5f)&&(abs(current_position.pose.position.y-pose.pose.position.y)<0.5f)&&(abs(current_position.pose.position.y-pose.pose.position.y)<0.5f))
           {
               pose.pose.position.x += 5;
               pose.pose.position.y = 20*sin(pose.pose.position.x/40*PI);
               pose.pose.position.z = 3;
           }
   
           local_pos_pub.publish(pose);
   
           ros::spinOnce();
           rate.sleep();
       }
   
       return 0;
   }
   ```

   其中通过订阅`mavros`发布的的`mavros/local_position/pose`话题得到当前无人机的位置反馈

   6.修改CMakeLists.txt文件，在最后加入下面两行：

   ```
   add_executable(offboard_node src/offboard_node.cpp)
   target_link_libraries(offboard_node ${catkin_LIBRARIES})
   ```

   7.打开一个terminal，运行外部控制节点

   `rosrun offboard offboard_node`

   即可看到仿真开始运行

## 四、真机部署

### 1.无人机搭建

| 指标       | 参数              |
| ---------- | ----------------- |
| 机架型号   | Z410              |
| 电机型号   | 朗宇 A2212        |
| 电子调速器 | 好盈 20A 长线版   |
| 供电系统   | 3S 4000mAh        |
| 遥控器     | 富斯I6X           |
| 飞行控制器 | Pixhawk 2.4.8     |
| 视觉里程计 | Intel T265        |
| 激光传感器 | 北醒 TF mini Plus |
| 数传模块   |                   |
| 机载处理器 | 树莓派4B 8GB      |

### 2.Z410安装

以阿木官方视频为主

将T265、Pixhwak2.4.8、机载电脑、电池等元器件合理布局于Z410机架之上。

https://www.bilibili.com/video/BV1JU4y1w77g?spm_id_from=333.337.search-card.all.click&vd_source=7b007210fdd20d8763f0ff79c3bea715

### 3.真机起飞

之前文章当中介绍了如何搭建ROS系统，并且运行MAVROS包来监听一个飞控消息，本篇文章将讨论如何使用MAVROS来控制无人机进行一键起飞功能。

#### 3.1创建一个offboard节点

##### *1、在/catkin_ws/src目录中创建一个新的offb_node.cpp文件*

```
roscd px4_mavros/src
vim offb_node.cpp
```

##### *2、写入控制代码*	

 参考官网给出的历程	

```c++
/**
 * @file offb_node.cpp
 * @brief Offboard control example node, written with MAVROS version 0.19.x, PX4 Pro Flight
 * Stack and tested in Gazebo SITL
 */

#include <ros/ros.h>
#include <geometry_msgs/PoseStamped.h>
#include <mavros_msgs/CommandBool.h>
#include <mavros_msgs/SetMode.h>
#include <mavros_msgs/State.h>

mavros_msgs::State current_state;
void state_cb(const mavros_msgs::State::ConstPtr& msg){
    current_state = *msg;
}

int main(int argc, char **argv)
{
    ros::init(argc, argv, "offb_node");
    ros::NodeHandle nh;

    ros::Subscriber state_sub = nh.subscribe<mavros_msgs::State>
            ("mavros/state", 10, state_cb);
    ros::Publisher local_pos_pub = nh.advertise<geometry_msgs::PoseStamped>
            ("mavros/setpoint_position/local", 10);
    ros::ServiceClient arming_client = nh.serviceClient<mavros_msgs::CommandBool>
            ("mavros/cmd/arming");
    ros::ServiceClient set_mode_client = nh.serviceClient<mavros_msgs::SetMode>
            ("mavros/set_mode");

    //the setpoint publishing rate MUST be faster than 2Hz
    ros::Rate rate(20.0);

  	  // wait for FCU connection
    while(ros::ok() && !current_state.connected){
        ros::spinOnce();
        rate.sleep();
    }

    geometry_msgs::PoseStamped pose;
    pose.pose.position.x = 0;
    pose.pose.position.y = 0;
    pose.pose.position.z = 2;

    //send a few setpoints before starting
    for(int i = 100; ros::ok() && i > 0; --i){
        local_pos_pub.publish(pose);
        ros::spinOnce();
        rate.sleep();
    }

    mavros_msgs::SetMode offb_set_mode;
    offb_set_mode.request.custom_mode = "OFFBOARD";

    mavros_msgs::CommandBool arm_cmd;
    arm_cmd.request.value = true;

    ros::Time last_request = ros::Time::now();

    while(ros::ok()){
        if( current_state.mode != "OFFBOARD" &&
            (ros::Time::now() - last_request > ros::Duration(5.0))){
            if( set_mode_client.call(offb_set_mode) &&
                offb_set_mode.response.mode_sent){
                ROS_INFO("Offboard enabled");
            }
            last_request = ros::Time::now();
        } else {
            if( !current_state.armed &&
                (ros::Time::now() - last_request > ros::Duration(5.0))){
                if( arming_client.call(arm_cmd) &&
                    arm_cmd.response.success){
                    ROS_INFO("Vehicle armed");
                }
                last_request = ros::Time::now();
            }
        }

        local_pos_pub.publish(pose);

        ros::spinOnce();
        rate.sleep();
    }

    return 0;
}
```

##### *3、编辑CMakeLists.txt文件*

```
rosed px4_mavros CMakeLists.txt
```

在CMakeLists.txt加入一下代码：

```
add_executable(offb_node src/offb_node.cpp)

target_link_libraries(offb_node ${catkin_LIBRARIES})
```

##### ***4、编译源码***

```	
cd ~/catkin_ws

catkin_make
```

#### 3.2运行offboard节点

警告：在进行下面操作前请将螺旋桨摘除再进行测试，避免误伤。

##### *1、运行启动文件* 

```undefined
roslaunch mavros px4.launch
```

##### ***2、启动按钮解锁***

 长按启动安全开关进行解锁操作。

##### ***3、运行节点***

 如果有仿真环境，则可以看到飞行器缓缓升高，直至保持在一个高度。

`rosrun px4_mavros offb_node`



## 五、简单例程

请关注阿木实验室的普罗米修斯项目(c++)和肖昆老师的XTDrone(python)项目，本文只提供入门指导以及简单指点飞行例程的跑通。

PX4 支持的模式如下：http://wiki.ros.org/mavros/CustomModes#PX4_native_flight_stack



仿真前的主机工作

起飞前的树莓派工作

```C++
roslaunch realsense2_camera rs_t265.launch
roslaunch mavros px4.launch
roslaunch vision_to_mavros t265_tf_to_mavros.launch
    、
rostopic echo mavros/vision_pose/pose
    
位置数据正常输出且可以稳定切换到定点，则可以起飞
R
```



### 1.指点飞行

```C++
/**
 * @file offb_node.cpp
 * @brief Offboard control example node, written with MAVROS version 0.19.x, PX4 Pro Flight
 * Stack and tested in Gazebo SITL
 */

#include <ros/ros.h>
#include <geometry_msgs/PoseStamped.h>
#include <mavros_msgs/CommandBool.h>
#include <mavros_msgs/SetMode.h>
#include <mavros_msgs/State.h>
#include <mavros_msgs/PositionTarget.h>

mavros_msgs::State current_state;
void state_cb(const mavros_msgs::State::ConstPtr& msg){
    current_state = *msg;
}

int pointFlag;
int main(int argc, char **argv)
{
    ros::init(argc, argv, "offb_node");
    ros::NodeHandle nh;

    ros::Subscriber state_sub = nh.subscribe<mavros_msgs::State>
            ("mavros/state", 10, state_cb);
    ros::Publisher local_pos_pub = nh.advertise<geometry_msgs::PoseStamped>
            ("mavros/setpoint_position/local", 10);
    ros::ServiceClient arming_client = nh.serviceClient<mavros_msgs::CommandBool>
            ("mavros/cmd/arming");
    ros::ServiceClient set_mode_client = nh.serviceClient<mavros_msgs::SetMode>
            ("mavros/set_mode");

    //the setpoint publishing rate MUST be faster than 2Hz
    ros::Rate rate(20.0);

    // wait for FCU connection
    while(ros::ok() && !current_state.connected){
        ros::spinOnce();
        rate.sleep();
    }

    geometry_msgs::PoseStamped pose;
   

    //send a few setpoints before starting
    for(int i = 100; ros::ok() && i > 0; --i){
        local_pos_pub.publish(pose);
        ros::spinOnce();
        rate.sleep();
    }

    mavros_msgs::SetMode offb_set_mode;
    offb_set_mode.request.custom_mode = "OFFBOARD";

    mavros_msgs::CommandBool arm_cmd;
    arm_cmd.request.value = true;

    ros::Time last_request = ros::Time::now();
	pointFlag = 0;// 初始化点位 0;
    while(ros::ok()){
        if( current_state.mode != "OFFBOARD" &&
            (ros::Time::now() - last_request > ros::Duration(5.0))){
            if( set_mode_client.call(offb_set_mode) &&
                offb_set_mode.response.mode·_sent){
                ROS_INFO("Offboard enabled");
            }
            last_request = ros::Time::now();
        } else {
            if( !current_state.armed &&
                (ros::Time::now() - last_request > ros::Duration(5.0))){
                if( arming_client.call(arm_cmd) &&
                    arm_cmd.response.success){
                    ROS_INFO("Vehicle armed");
                }
                last_request = ros::Time::now();
            }
        }
		if(ros::Time::now() - last_request > ros::Duration(5)){
            switch(pointFlag){
            case 0:{
                pose.pose.position.x = 0;
   				pose.pose.position.y = 0;
    		 	pose.pose.position.z = 2;
                last_request = ros::Time::now;
            }
            case 1: {
                 pose.pose.position.x = 1;
   				 pose.pose.position.y = 1;
    		 	 pose.pose.position.z = 2;
                 last_request = ros::Time::now;
            }
            case 3:{
                 pose.pose.position.x = 2;
   				 pose.pose.position.y = 2;
    		 	 pose.pose.position.z = 2;
                 last_request = ros::Time::now;
            }
       	 }
            pointFlag++;
        }
        
        local_pos_pub.publish(pose);

        ros::spinOnce();
        rate.sleep();
    }

    return 0;
}
```

### 2.实现offboard模式sin路线飞行控制

1. 找到PX4的安装路径，打开gazebo仿真并启动PX4：

   ```
   cd Firmware
   make px4_sitl gazebo_iris
   ```

2. 打开QGC地面站

3. 建立mavros与PX4的连接

   ```
   roslaunch mavros px4.launch fcu_url:="udp://:14540@127.0.0.1:14557"
   ```

4. 找到ROS工作空间，创建功能包

   ```
   catkin_create_pkg offboard_sin roscpp std_msgs geometry_msgs mavros_msgs
   ```

   在功能包的src路径下创建offboard_sin_node.cpp文件写入如下代码

   ```C++
   /**
    * @file offb_node.cpp
    * @brief Offboard control example node, written with MAVROS version 0.19.x, PX4 Pro Flight
    * Stack and tested in Gazebo SITL
    */
   
   #include <ros/ros.h>
   #include <geometry_msgs/PoseStamped.h>
   #include <geometry_msgs/Vector3.h>
   #include <mavros_msgs/CommandBool.h>
   #include <mavros_msgs/SetMode.h>
   #include <mavros_msgs/State.h>
   
   #define PI acos(-1)
   
   mavros_msgs::State current_state;
   geometry_msgs::PoseStamped current_position;
   
   void state_cb(const mavros_msgs::State::ConstPtr& msg){
       current_state = *msg;
   }
   void getpointfdb(const geometry_msgs::PoseStamped::ConstPtr& msg){
       ROS_INFO("x: [%f]", msg->pose.position.x);
       ROS_INFO("y: [%f]", msg->pose.position.y);
       ROS_INFO("z: [%f]", msg->pose.position.z);
       current_position = *msg;
   }
   
   int main(int argc, char **argv)
   {
       ros::init(argc, argv, "offb_node");
       ros::NodeHandle nh;
   
       ros::Subscriber state_sub = nh.subscribe<mavros_msgs::State>
               ("mavros/state", 10, state_cb);
               
       ros::Subscriber get_point = nh.subscribe<geometry_msgs::PoseStamped>
               ("mavros/local_position/pose", 10, getpointfdb);
               
       ros::Publisher local_pos_pub = nh.advertise<geometry_msgs::PoseStamped>
               ("mavros/setpoint_position/local", 10);
       ros::ServiceClient arming_client = nh.serviceClient<mavros_msgs::CommandBool>
               ("mavros/cmd/arming");
       ros::ServiceClient set_mode_client = nh.serviceClient<mavros_msgs::SetMode>
               ("mavros/set_mode");
   
       //the setpoint publishing rate MUST be faster than 2Hz
       ros::Rate rate(20.0f);
   
       // wait for FCU connection
       while(ros::ok() && !current_state.connected){
           ros::spinOnce();
           rate.sleep();
       }
   
       geometry_msgs::PoseStamped pose;
       pose.pose.position.x = 0;
       pose.pose.position.y = 0;
       pose.pose.position.z = 3;
       
       
       //send a few setpoints before starting
       for(int i = 100; ros::ok() && i > 0; --i){
           local_pos_pub.publish(pose);
           ros::spinOnce();
           rate.sleep();
       }
   
       mavros_msgs::SetMode offb_set_mode;
       offb_set_mode.request.custom_mode = "OFFBOARD";
   
       mavros_msgs::CommandBool arm_cmd;
       arm_cmd.request.value = true;
   
       ros::Time last_request = ros::Time::now();
   
       while(ros::ok()){
           if( current_state.mode != "OFFBOARD" &&
               (ros::Time::now() - last_request > ros::Duration(5.0f))){
               if( set_mode_client.call(offb_set_mode) &&
                   offb_set_mode.response.mode_sent){
                   ROS_INFO("Offboard enabled");
               }
               last_request = ros::Time::now();
           } else {
               if( !current_state.armed &&
                   (ros::Time::now() - last_request > ros::Duration(5.0f))){
                   if( arming_client.call(arm_cmd) &&
                       arm_cmd.response.success){
                       ROS_INFO("Vehicle armed");
                   }
                   last_request = ros::Time::now();
               }
           }
           
           if((abs(current_position.pose.position.x-pose.pose.position.x)<0.5f)&&(abs(current_position.pose.position.y-pose.pose.position.y)<0.5f)&&(abs(current_position.pose.position.y-pose.pose.position.y)<0.5f))
           {
               pose.pose.position.x += 5;
               pose.pose.position.y = 20*sin(pose.pose.position.x/40*PI);
               pose.pose.position.z = 3;
           }
   
           local_pos_pub.publish(pose);
   
           ros::spinOnce();
           rate.sleep();
       }
   
       return 0;
   }
   ```

   其中通过订阅`mavros`发布的的`mavros/local_position/pose`话题得到当前无人机的位置反馈

5. 修改CMakeLists.txt文件，在最后加入下面两行：

   ```
   add_executable(offboard_node src/offboard_node.cpp)
   target_link_libraries(offboard_node ${catkin_LIBRARIES})
   ```

   最终实现效果:

![在这里插入图片描述](%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B.assets/20210717104016796.gif)







# 机载计算机控制APM固件飞控的缺点

## APM固件：

机载计算机通过mavlink控制apm飞控的能力有限。经过尝试，mavros控制apm飞控情况如下
**目前只支持：**

* 三个姿态角和推力（需要自己写姿态控制环），且所有姿态角和推力必须同时给出(setpoint_raw/attitude)
* ‌在飞机靠遥控飞起来后，控制三个方向的速度，且xyz速度需要同时给出(setpoint_velocity/cmd_vel)
* 也许还可以支持飞起来之后给三个方向的位置（没试成功）(setpoint_postition/local)

不支持：

* zebo仿真
* ‌加速度控制
* ‌角速度控制
* 从怠速开始guided控制（这意味着无法自动起飞）
* ‌同时控制速度，高度。或者同时控制位置，航向角等，不支持setpoint_raw/local
* **自动广播飞控姿态和位置**，速度。需要输入rosservice命令手动打开

## px4固件：

机载计算机利用mavros控制px4固件：
优势：

* 持位置，加速度，速度，角速度，姿态角和推力控制以及上述部分组合的混合控制
* ‌支持gazebo仿真
* ‌自动广播飞控数据
* ‌可以offboard模式下解锁，切换模式，能自动起降
* ‌px4固件支持北醒tfmini plus激光定高，设置方便

待解决：

* 不支持nmea的gps协议，因此不能使用nooploop的伪gps直接连接飞控（但是可以通过机载计算机向飞控发送uwb位置来解决）
* ‌加速度控制性能未知
* ‌飞控发布的速度准确度未知

总之，APM飞控完全无法胜任我们对机载计算机控制飞行的要求，不仅不能利用机载计算机实现加速度控制、位置控制、位置速度混合控制等，还要求飞机起飞后再切换guided模式、不自动广播飞控数据，也不支持gazebo仿真。

px4显然是更加契合ros，gazebo以及机载计算机控制的飞控固件。





# PX4固件调参

机载电脑参数配置:

T265参数配置：

| 参数                                                         | 外部位置估计的设置                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [EKF2_AID_MASK](https://docs.px4.io/main/zh/advanced_config/parameter_reference.html#EKF2_AID_MASK) | Set *vision position fusion*, *vision velocity fusion*, *vision yaw fusion* and *external vision rotation* according to your desired fusion model. |
| [EKF2_HGT_MODE](https://docs.px4.io/main/zh/advanced_config/parameter_reference.html#EKF2_HGT_MODE) | 设置为 *Vision* 使用视觉作为高度估计的主要来源。             |
| [EKF2_EV_DELAY](https://docs.px4.io/main/zh/advanced_config/parameter_reference.html#EKF2_EV_DELAY) | 设置为测量的时间戳和 "实际" 捕获时间之间的差异。 有关详细信息，请参阅 [below](https://docs.px4.io/main/zh/computer_vision/visual_inertial_odometry.html#tuning-EKF2_EV_DELAY)。 |
| [EKF2_EV_POS_X](https://docs.px4.io/main/zh/advanced/parameter_reference.html#EKF2_EV_POS_X), [EKF2_EV_POS_Y](https://docs.px4.io/main/zh/advanced/parameter_reference.html#EKF2_EV_POS_Y), [EKF2_EV_POS_Z](https://docs.px4.io/main/zh/advanced/parameter_reference.html#EKF2_EV_POS_Z) | 设置视觉传感器相对于车身框架的位置。                         |









# Px4_command

https://gitee.com/theroadofengineers/px4_command

## 简介

px4_command功能包是一个基于PX4开源固件及Mavros功能包的开源项目，旨在为PX4开发者提供更加简洁快速的开发体验。目前已集成无人机外环控制器修改、目标追踪及避障等上层开发代码，后续将陆续推出涵盖任务决策、路径规划、滤波导航、单/多机控制等无人机科研及开发领域的功能。

## 安装

1. 通过二进制的方法安装Mavros功能包

   请参考: [https://github.com/mavlink/mavros](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Fmavlink%2Fmavros)

   如果你已经使用源码的方式安装过Mavros功能包，请先将其删除。

2. 在home目录下创建一个名为 "px4_ws" 的工作空间

   `mkdir -p ~/px4_ws/src`

   `cd ~/px4_ws/src`

   `catkin_init_workspace`

   大部分时候，需要手动进行source，打开一个新终端

   `gedit .bashrc`

   在打开的`bashrc.txt`文件中添加 `source /home/$(your computer name)/px4_ws/devel/setup.bash`

3. 下载并编译 `px4_command` 功能包

   `cd ~/px4_ws/src`

   `git clone https://gitee.com/theroadofengineers/px4_command`

   `cd ..`

   `catkin_make`

## 项目总览

![未命名表单](https://camo.githubusercontent.com/22e99b831d0c4b567dd2d8cac4850d8c44d6c248aa254b4d50e7d0893ff977d9/68747470733a2f2f692e696d6775722e636f6d2f417536394770732e706e67)

- 读取飞控状态 [state_from_mavros.h](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Finclude%2Fstate_from_mavros.h)
- 发送控制指令至飞控 [command_to_mavros.h](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Finclude%2Fcommand_to_mavros.h)
- 位置环控制器实现（目前提供五种外环控制器，分别为串级PID、PID、UDE、Passivity+UDE、NE+UDE） [pos_controller_cascade_PID.h](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Finclude%2Fpos_controller_cascade_PID.h) [pos_controller_PID.h](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Finclude%2Fpos_controller_PID.h) [pos_controller_UDE.h](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Finclude%2Fpos_controller_UDE.h) [pos_controller_Passivity.h](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Finclude%2Fpos_controller_Passivity.h) [pos_controller_NE.h](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Finclude%2Fpos_controller_NE.h) 其中，串级PID为仿写PX4中位置控制器、Passivity+UDE为无需速度反馈的位置控制器、NE+UDE在速度测量有噪声时由于其他控制器。
- 外部定位实现 [px4_pos_estimator.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2Fpx4_pos_estimator.cpp)
- 控制逻辑主程序 [px4_pos_controller.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2Fpx4_pos_controller.cpp)
- 地面站（需配合ROS多机使用） [ground_station.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2Fground_station.cpp)
- 自主降落 [autonomous_landing.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2FApplication%2Fautonomous_landing.cpp)
- 简易避障 [collision_avoidance.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2FApplication%2Fcollision_avoidance.cpp)
- 双目简易避障 [collision_avoidance_streo.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2FApplication%2Fcollision_avoidance_streo.cpp)
- 编队飞行（目前仅支持gazebo仿真）[formation_control_sitl.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2FApplication%2Fformation_control_sitl.cpp)
- 负载投掷 [payload_drop.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2FApplication%2Fpayload_drop.cpp)
- 航点追踪 [square.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2FApplication%2Fsquare.cpp)
- 目标追踪 [target_tracking.cpp](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2FApplication%2Ftarget_tracking.cpp)

> 说明： 1、其中自主降落、目标追踪、双目简易避障需配合vision部分代码使用。 2、功能包中还包含一些简易滤波及测试小代码，此处不作说明，可自行查阅。

## 视频演示

[自主降落](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2Fav60648116%2F)

[负载投掷](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2Fav55037908%2F)

[简易避障、目标追踪](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2Fav60648886%2F)

[外环控制器修改 NE+UDE](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2Fav60963113%2F) 及 [外环控制器修改 Passivity-based UDE](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2Fav60979252%2F)

[内环控制器修改（PX4固件）](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2Fav60962814%2F)

## 坐标系说明

本功能包中所有变量均为 **ENU** 坐标系（同Mavros，异于PX4固件）

> MAVROS does translate Aerospace NED frames, used in FCUs to ROS ENU frames and vice-versa. For translate airframe related data we simply apply rotation 180° about ROLL (X) axis. For local we apply 180° about ROLL (X) and 90° about YAW (Z) axes

## PX4固件及参数

请使用提供代码中提供的PX4固件 - [firmware](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Ftree%2Fmaster%2Ffirmware)

SYS_COMPANION参数设置为Companion（921600）。

若需要使用外部定位，则参数EKF2_AID_MASK设为24（默认为1），EKF2_HGT_MODE设置为VISION（默认为气压计）。

## 其他教程

[随意写写](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Fpotato77%2FTech_Blog)

## 多机Gazebo仿真

多机仿真前，请确保PX4固件能够顺利运行单机及双机仿真例程。

请先将该文件[iris_3](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsrc%2FApplication%2Firis_3)放置于PX4固件Firmware/posix-configs/SITL/init/ekf2/目录下（固件版本 v1.8.2），然后运行脚本[sitl_gazebo_formation](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Famov-lab%2Fpx4_command%2Fblob%2Fmaster%2Fsh%2Fsh_for_simulation%2Fsitl_gazebo_formation.sh)即可。

# 远程桌面

推荐使用[nomachine](https://gitee.com/link?target=https%3A%2F%2Fwww.nomachine.com)作为远程桌面。



​	
