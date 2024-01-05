## d435i双目+IMU联合标定

### 1. build Kalibr from source code

1. install ROS1 on system, for example

```bash
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
sudo apt-get update
export ROS1_DISTRO=noetic # melodic=18.04, noetic=20.04
sudo apt-get install ros-$ROS1_DISTRO-desktop-full
sudo apt-get install python-catkin-tools # ubuntu 18.04
sudo apt-get install python3-catkin-tools python3-osrf-pycommon # ubuntu 20.04

#酌情根据需要安装依赖项（optional）
sudo apt-get install -y \
    git wget autoconf automake nano \
    libeigen3-dev libboost-all-dev libsuitesparse-dev \
    doxygen libopencv-dev \
    libpoco-dev libtbb-dev libblas-dev liblapack-dev libv4l-dev

#Then due to different Python versions, you will need to install the following:
# Ubuntu 18.04
sudo apt-get install -y python3-dev python-pip python-scipy \
    python-matplotlib ipython python-wxgtk4.0 python-tk python-igraph python-pyx
# Ubuntu 20.04
sudo apt-get install -y python3-dev python3-pip python3-scipy \
    python3-matplotlib ipython3 python3-wxgtk4.0 python3-tk python3-igraph python3-pyx
```

2. create a catkin workspace and clone the kalibr project

```bash
mkdir -p ~/kalibr/src
cd ~/kalibr
export ROS1_DISTRO=noetic # kinetic=16.04, melodic=18.04, noetic=20.04
source /opt/ros/$ROS1_DISTRO/setup.bash
catkin init
catkin config --extend /opt/ros/$ROS1_DISTRO
catkin config --merge-devel # Necessary for catkin_tools >= 0.4.
catkin config --cmake-args -DCMAKE_BUILD_TYPE=Release
#then clone the project
cd ~/kalibr/src
git clone https://github.com/ethz-asl/kalibr.git
```

3. build

```bash
cd ~/kalibr
#根据需要修改j数值
catkin build -DCMAKE_BUILD_TYPE=Release -j14
```

4. source the catkin workspace setup to use Kalibr，至此**Kalibr**安装完成

```bash
source ~/kalibr/devel/setup.bash
rosrun kalibr <command_you_want_to_run_here>
```

5. 安装**code_utils** & **imu_utils** 工具用于后面标定IMU

```bash
#先安装ceres
# google-glog + gflags
sudo apt-get install libgoogle-glog-dev libgflags-dev
# now ready to build, test, and install Ceres.
# download ceres-solver-2.2.0 from https://github.com/ceres-solver/ceres-solver/archive/refs/tags/2.2.0.tar.gz
tar zxf ceres-solver-2.2.0.tar.gz
mkdir ceres-bin
cd ceres-bin
cmake ../ceres-solver-2.2.0
make -j14
make test
# Optionally install Ceres, it can also be exported using CMake which
# allows Ceres to be used without requiring installation, see the documentation
# for the EXPORT_BUILD_DIR option for more information.
sudo make install

#安装code utils
sudo apt install libdw-dev
mkdir -p ~/imu_calib/src
cd ~/imu_calib/src
catkin_init_workspace
git clone https://github.com/silencht/code_utils.git
#仓库文件已经将sumpixel_test.cpp中#include "backward.hpp"改为：#include "code_utils/backward.hpp"
#如果下方步骤编译出错，可酌情再次修改为#include "backward.hpp"
catkin_make -j14

#安装imu utils
cd ~/imu_calib/src/
git clone https://github.com/silencht/imu_utils.git
cd ..
catkin_make -j14
```

6. 为d435i相机安装realsense ros1驱动包

```bash
#https://github.com/IntelRealSense/realsense-ros/tree/ros1-legacy
#create a catkin workspace for realsense ros
mkdir -p ~/d435i_ros/src
cd ~/d435i_ros/src
git clone https://github.com/silencht/realsense-ros.git
cd realsense-ros/
git checkout `git tag | sort -V | grep -P "^2.\d+\.\d+" | tail -1`
cd ..
catkin_init_workspace
cd ..
catkin_make clean
catkin_make -DCATKIN_ENABLE_TESTING=False -DCMAKE_BUILD_TYPE=Release
catkin_make install
#运行和测试查看
roslaunch realsense2_camera rs_camera.launch
rviz
```



### 2. IMU标定

```bash
#rs_camera_for_calib.launch主要相对于rs_camera.launch修改了
#  <arg name="infra_width"         default="640"/>       此行作用：将红外图像分辨率由848*480变为640*480
#  <arg name="gyro_fps"            default="400"/>
#  <arg name="accel_fps"           default="250"/>
#  <arg name="enable_gyro"         default="true"/>
#  <arg name="enable_accel"        default="true"/>
#  <arg name="unite_imu_method"    default="linear_interpolation"/>
#启动相机驱动程序，开始发布数据
roslaunch realsense2_camera rs_camera_for_calib.launch
#此时相机开始输出数据，其中有imu数据，可使用如下命令查看数据是否正常发布
changhe@changhe:~/d435i_ros$ rostopic list -v | grep imu
 * /camera/imu [sensor_msgs/Imu] 1 publisher
 * /camera/gyro/imu_info [realsense2_camera/IMUInfo] 1 publisher
 * /camera/accel/imu_info [realsense2_camera/IMUInfo] 1 publisher
changhe@changhe:~/d435i_ros$ rostopic echo /camera/imu
header: 
  seq: 10135
  stamp: 
    secs: 1704355811
    nsecs: 680399179
  frame_id: "camera_imu_optical_frame"
orientation: 
  x: 0.0
  y: 0.0
  z: 0.0
  w: 0.0
orientation_covariance: [-1.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
angular_velocity: 
  x: -0.001745329238474369
  y: 0.001745329238474369
  z: 0.0
angular_velocity_covariance: [0.01, 0.0, 0.0, 0.0, 0.01, 0.0, 0.0, 0.0, 0.01]
linear_acceleration: 
  x: -0.00493930911018656
  y: -9.846020630401458
  z: 0.019613299518823624
linear_acceleration_covariance: [0.01, 0.0, 0.0, 0.0, 0.01, 0.0, 0.0, 0.0, 0.01]

#静置相机，录制2小时以上的ROS bag，2小时以后使用ctrl+c终止命令，imu_calib目录下产生imu_calibration.bag文件
#或者也可以跳过该步骤，直接使用静置相机的话题发布数据，不记录ROS bag
cd ~/imu_calib
rosbag record -O imu_calibration /camera/imu

#运行IMU校准程序，如果认为2小时太长，可以在d435i_imu_calibration.launch中max_time_min设置value调整计时分钟
source ~/imu_calib/devel/setup.bash
roslaunch imu_utils d435i_imu_calibration.launch
#执行上述命令后，输出以下信息，便可等待程序执行完成（2小时以后）
[ INFO] [1704356481.837577493]: Loaded imu_topic: /camera/imu
[ INFO] [1704356481.838011572]: Loaded imu_name: d435i
[ INFO] [1704356481.838159386]: Loaded data_save_path: /home/changhe/imu_calib/src/imu_utils/data/
[ INFO] [1704356481.838308947]: Loaded max_time_min: 120
[ INFO] [1704356481.838460625]: Loaded max_cluster: 400
gyr x  num of Cluster 400
gyr y  num of Cluster 400
gyr z  num of Cluster 400
acc x  num of Cluster 400
acc y  num of Cluster 400
acc z  num of Cluster 400
wait for imu data.

#打开ROS bag数据包进行回放，如果使用相机的直出发布数据则忽略该步骤
rosbag play -r 400 imu_calibration.bag

#标定结束后，标定结果位于imu_utils/data目录下的d435i_imu_param.yaml文件内,终端输出示例：
wait for imu data.
gyr x  numData 2878675
gyr x  start_t 1.70436e+09
gyr x  end_t 1.70436e+09
gyr x dt 
-------------7200.01 s
-------------120 min
-------------2 h
gyr x  freq 399.816
gyr x  period 0.00250115
gyr y  numData 2878675
gyr y  start_t 1.70436e+09
gyr y  end_t 1.70436e+09
gyr y dt 
-------------7200.01 s
-------------120 min
-------------2 h
gyr y  freq 399.816
gyr y  period 0.00250115
gyr z  numData 2878675
gyr z  start_t 1.70436e+09
gyr z  end_t 1.70436e+09
gyr z dt 
-------------7200.01 s
-------------120 min
-------------2 h
gyr z  freq 399.816
gyr z  period 0.00250115
Gyro X 
C    -2.87589     70.9924    -7.40893    0.642781 -0.00746316
 Bias Instability 4.1718e-06 rad/s
 Bias Instability 2.14632e-05 rad/s, at 361.957 s
 White Noise 21.8314 rad/s
 White Noise 0.00606455 rad/s
  bias -0.15276 degree/s
-------------------
Gyro y 
C   -2.85711    70.9111   -5.49533   0.648136 -0.0109878
 Bias Instability 1.40587e-05 rad/s
 Bias Instability 1.93153e-05 rad/s, at 609.529 s
 White Noise 22.6084 rad/s
 White Noise 0.0062829 rad/s
  bias 0.0119102 degree/s
-------------------
Gyro z 
C   -0.975676     27.4672    -4.80227    0.445228 -0.00711166
 Bias Instability 1.5707e-06 rad/s
 Bias Instability 1.59334e-05 rad/s, at 87.0977 s
 White Noise 7.53363 rad/s
 White Noise 0.00209385 rad/s
  bias -0.0410812 degree/s
-------------------
==============================================
==============================================
acc x  numData 2878675
acc x  start_t 1.70436e+09
acc x  end_t 1.70436e+09
acc x dt 
-------------7200.01 s
-------------120 min
-------------2 h
acc x  freq 399.816
acc x  period 0.00250115
acc y  numData 2878675
acc y  start_t 1.70436e+09
acc y  end_t 1.70436e+09
acc y dt 
-------------7200.01 s
-------------120 min
-------------2 h
acc y  freq 399.816
acc y  period 0.00250115
acc z  numData 2878675
acc z  start_t 1.70436e+09
acc z  end_t 1.70436e+09
acc z dt 
-------------7200.01 s
-------------120 min
-------------2 h
acc z  freq 399.816
acc z  period 0.00250115
acc X 
C -4.98469e-06  0.000757756   3.7721e-05  2.22768e-05 -3.97843e-07
 Bias Instability 0.000280485 m/s^2
 White Noise 0.0162395 m/s^2
-------------------
acc y 
C -2.49845e-07  0.000834302 -1.96216e-05  1.89497e-05 -3.22867e-07
 Bias Instability 0.000192201 m/s^2
 White Noise 0.0171629 m/s^2
-------------------
acc z 
C -2.81212e-06  0.000594583  6.58393e-05  6.43312e-06 -1.03843e-07
 Bias Instability 0.000179308 m/s^2
 White Noise 0.0128838 m/s^2
-------------------
[imu_an-1] process has finished cleanly
log file: /home/changhe/.ros/log/965a1072-aad8-11ee-b508-a13a0030ff2c/imu_an-1*.log
all processes on machine have died, roslaunch will exit
shutting down processing monitor...
... shutting down processing monitor complete
done

#d435i_imu_param.yaml文件内容示例
%YAML:1.0
---
type: IMU
name: d435i
Gyr:
   unit: " rad/s"
   avg-axis:
      gyr_n: 4.8137652096146783e-03
      gyr_w: 1.8903981803128362e-05
   x-axis:
      gyr_n: 6.0645500436790623e-03
      gyr_w: 2.1463185558878976e-05
   y-axis:
      gyr_n: 6.2828973927536100e-03
      gyr_w: 1.9315340333546505e-05
   z-axis:
      gyr_n: 2.0938481924113613e-03
      gyr_w: 1.5933419516959604e-05
Acc:
   unit: " m/s^2"
   avg-axis:
      acc_n: 1.5428717790802770e-02
      acc_w: 2.1733118964279978e-04
   x-axis:
      acc_n: 1.6239510893649358e-02
      acc_w: 2.8048486828980107e-04
   y-axis:
      acc_n: 1.7162885741267130e-02
      acc_w: 1.9220055801685360e-04
   z-axis:
      acc_n: 1.2883756737491822e-02
      acc_w: 1.7930814262174464e-04
```

### 3. 双目标定

1. 生成标定板并用A4纸打印

```bash
cd ~/kalibr
source devel/setup.sh
#--type apriltag                标定板类型
#--nx [NUM_COLS]                列个数
#--ny [NUM_ROWS]                行个数
#--tsize [TAG_WIDTH_M]          二维码方格长度，单位m
#--tspace [TAG_SPACING_PERCENT] 小方格与二维码方格长度比例
rosrun kalibr kalibr_create_target_pdf --type apriltag --nx 6 --ny 6 --tsize 0.022 --tspace 0.3
#新建april_6x6_A4.yaml文件，内容如下
target_type: 'aprilgrid' #gridtype
tagCols: 6               #number of apriltags
tagRows: 6               #number of apriltags
tagSize: 0.022           #size of apriltag, edge to edge [m]
tagSpacing: 0.3          #ratio of space between tags to tagSize
```

2. 关闭双目结构光发射器，避免图像产生斑点

```bash
roslaunch realsense2_camera rs_camera_for_calib.launch
rviz
rosrun rqt_reconfigure rqt_reconfigure
#将camera/stereo_module标签页右侧的emitter_enabled设置为Off(0)，即关闭发射器，此时rviz中的双目图像斑点消失
```

3. 【请忽略该步骤】限制相机帧数（[官方推荐4Hz](https://github.com/ethz-asl/kalibr/wiki/multiple-camera-calibration)）因为出现 Cameras are not connected through mutual observations, please check the dataset. Maybe adjust the approx. sync. tolerance.报错所以该步骤放弃。

```bash
rosrun topic_tools throttle messages /camera/infra1/image_rect_raw 4.0 /infra_left
rosrun topic_tools throttle messages /camera/infra2/image_rect_raw 4.0 /infra_right
#可以通过rostopic hz /infra_left命令或者直接在rviz中检查
```

4. 录制ROS bag

```bash
cd ~/kalibr
rosbag record -O multicameras_calibration /camera/infra1/image_rect_raw /camera/infra2/image_rect_raw
#官方推荐标定板对着相机动，但是由于标定板是A4纸比较软，所以这里反过来(实际官方视频也是这么做的：https://www.youtube.com/watch?app=desktop&v=puNXsnrYWTY)
#运动的方式为：先绕相机的各自三个轴，每个轴旋转三次，然后再沿着相机的三个轴，每个轴平移三次，基本就可以了，运动期间要保证相机基本能一直看到标定板的全部信息。
#录制完毕后kalibr目录下生成multicameras_calibration.bag文件
#可通过rosbag info multicameras_calibration.bag查看数据包信息,示例如下：
changhe@changhe:~/kalibr$ rosbag info multicameras_calibration.bag 
path:        multicameras_calibration.bag
version:     2.0
duration:    1:11s (71s)
start:       Jan 05 2024 09:44:41.37 (1704419081.37)
end:         Jan 05 2024 09:45:52.49 (1704419152.49)
size:        1.6 GB
messages:    4268
compression: none [2134/2134 chunks]
types:       sensor_msgs/Image [060021388200f6f0f447d0fcd9c64743]
topics:      /camera/infra1/image_rect_raw   2134 msgs    : sensor_msgs/Image
             /camera/infra2/image_rect_raw   2134 msgs    : sensor_msgs/Image
```

5. 使用**Kalibr**进行双目标定

[Reprojection errors should be in a normal range (0.1-0.2 px for a good calibration)](https://github.com/ethz-asl/kalibr/wiki/calibrating-the-vi-sensor)

```bash
cd ~/kalibr
source devel/setup.bash
#注：april_6x6_A4.yaml是3.1节生成的
#--target：标定板参数信息存放的路径，即我们上面的april_s.yaml的路径
#--bag：录制的Camera数据包的路径
#--bag-from-to：起始和终止时间，单位是秒。为了避免开始录制和结束录制时的抖动，这里取了第3秒到第65秒之间的信息
#--models：相机模型，一般为针孔模型，https://github.com/ethz-asl/kalibr/wiki/supported-models
#--topics：影像流发布的话题
#--show-extraction：显示检测特征点的过程
rosrun kalibr kalibr_calibrate_cameras --target april_6x6_A4.yaml --bag multicameras_calibration.bag --bag-from-to 3 65 --models pinhole-radtan pinhole-radtan --topics /camera/infra1/image_rect_raw /camera/infra2/image_rect_raw --show-extraction
#执行成功后，生成三个文件，分别为：
1. multicameras_calibration-results-cam.txt
2. multicameras_calibration-camchain.yaml
3. multicameras_calibration-report-cam.pdf

#附：
#使用3.3步骤时，会出现双目图像流时间戳不同步问题，所以放弃3.3步骤，以下是相关issues以及可能的解决措施
#https://github.com/ethz-asl/kalibr/issues/364
#https://github.com/ethz-asl/kalibr/issues/332
#命令后添加 --approx-sync 0.04 将时间同步容忍度增加到0.04秒
rosrun kalibr kalibr_calibrate_cameras --target april_6x6_A4.yaml --bag multicameras_calibration.bag --bag-from-to 3 65 --models pinhole-radtan pinhole-radtan --topics /camera/infra1/image_rect_raw /camera/infra2/image_rect_raw --show-extraction --approx-sync 0.04
```



### 4. 双目&IMU联合标定

1. [新建camchain.yaml文件](https://github.com/ethz-asl/kalibr/wiki/yaml-formats)

```yaml
#根据3.5节得到的multicameras_calibration-camchain.yaml在~/kalibr目录创建camchain.yaml文件
#camchain.yaml内容参考如下
cam0:
  camera_model: pinhole
  intrinsics: [384.57401093793385, 384.77603230830897, 319.36432154640477, 236.5865796387184]
  distortion_model: radtan
  distortion_coeffs: [0.000868210041633816, -0.001536564418790415, -0.0008338819079702226, 0.0005407357906101209]
  rostopic: /camera/infra1/image_rect_raw
  resolution: [640, 480]
cam1:
  camera_model: pinhole
  intrinsics: [384.71257117832874, 385.0226535924584, 319.9598090030288, 236.36875214589423]
  distortion_model: radtan
  distortion_coeffs: [0.0027736443531086595, -0.004051540794976078, -0.0010880219876272697, 0.0006497706840705447]
  T_cn_cnm1:
  - [0.999997213819325, -6.564576770509914e-05, -0.0023596703620665503, -0.05009032712390797]
  - [6.840714919526547e-05, 0.9999993130009001, 0.0011701786988816156, 4.8368258872807713e-05]
  - [0.0023595919236960978, -0.0011703368568748496, 0.9999965313127803, -0.0001744522191402009]
  - [0.0, 0.0, 0.0, 1.0]
  rostopic: /camera/infra2/image_rect_raw
  resolution: [640, 480]
```

2. [新建imu.yaml文件](https://github.com/ethz-asl/kalibr/wiki/yaml-formats)

```bash
#根据第2节获得的d435i_imu_param.yaml文件在~/kalibr目录创建imu.yaml文件
#d435i_imu_param.yaml文件位于~/imu_calib/src/imu_utils/data/目录下
#imu.yaml内容参考如下：
#Accelerometers
accelerometer_noise_density: 1.5428717790802770e-02   #Noise density (continuous-time)
accelerometer_random_walk:   2.1733118964279978e-04   #Bias random walk

#Gyroscopes
gyroscope_noise_density:     4.8137652096146783e-03   #Noise density (continuous-time)
gyroscope_random_walk:       1.8903981803128362e-05   #Bias random walk

rostopic:                    /camera/imu       #the IMU ROS topic
update_rate:                 400.0             #Hz (for discretization of the values above)
```

3. 录制ROS bag数据

```bash
#~/d435i_ros/src/realsense-ros/realsense2_camera/launch/rs_camera_imu_for_calib.launch文件相比rs_camera_for_calib.launch
#改动了以下代码，将双目帧率调整至40hz，并打开了同步开关
#  <arg name="infra_fps"           default="40"/>
#  <arg name="enable_sync"   default="true"/>
#启动相机驱动
roslaunch realsense2_camera rs_camera_imu_for_calib.launch
#打开rviz，订阅双目话题，关闭结构光，仿照3.4节动作录制数据
rviz
rosrun rqt_reconfigure rqt_reconfigure
#将camera/stereo_module标签页右侧的emitter_enabled设置为Off(0)，即关闭发射器，此时rviz中的双目图像斑点消失
#开始录制数据包，注意拿起放下相机时的数据应尽可能排除，从而避免相机的抖动/振动
cd ~/kalibr/
rosbag record -O imu_stereo.bag /camera/infra1/image_rect_raw /camera/infra2/image_rect_raw /camera/imu
#录制完成后~/kalibr/目录下产生imu_stereo.bag
changhe@changhe:~/kalibr$ rosbag info imu_stereo.bag 
path:        imu_stereo.bag
version:     2.0
duration:    2:02s (122s)
start:       Jan 05 2024 12:12:07.82 (1704427927.82)
end:         Jan 05 2024 12:14:10.61 (1704428050.61)
size:        2.1 GB
messages:    56462
compression: none [2457/2457 chunks]
types:       sensor_msgs/Image [060021388200f6f0f447d0fcd9c64743]
             sensor_msgs/Imu   [6a62c6daae103f4ff57a132d6f95cec2]
topics:      /camera/imu                     49094 msgs    : sensor_msgs/Imu  
             /camera/infra1/image_rect_raw    3684 msgs    : sensor_msgs/Image
             /camera/infra2/image_rect_raw    3684 msgs    : sensor_msgs/Image
             
             
#附：
#由于rostopic hz topic_name检查时发现imu频率在125hz，很奇怪，故以下步骤跳过忽略（本文采取双目 40hz，imu 400hz的频率采集）
#cameras should run at 20 Hz and IMU at 200 Hz，from https://github.com/ethz-asl/kalibr/wiki/calibrating-the-vi-sensor
#检查频率可通过 rostopic hz topic_name
rosrun topic_tools throttle messages /camera/infra1/image_rect_raw 20.0 /infra_left & rosrun topic_tools throttle messages /camera/infra2/image_rect_raw 20.0 /infra_right & rosrun topic_tools throttle messages /camera/imu 200.0 /imu
```

4. 使用**Kalibr**进行双目+IMU联合标定

```bash
source  ~/kalibr/devel/setup.bash
#使用rosrun kalibr kalibr_calibrate_imu_camera启动标定程序
rosrun kalibr kalibr_calibrate_imu_camera --bag  imu_stereo.bag --cam  camchain.yaml --imu imu.yaml --target april_6x6_A4.yaml --bag-from-to 5 115 --show-extraction
#等待一段时间，获得结果，共四个文件
1. imu_stereo-camchain-imucam.yaml
2. imu_stereo-imu.yaml
3. imu_stereo-results-imucam.txt
4. imu_stereo-report-imucam.pdf
#注：reprojection errors should be in a normal range (0.1-0.2 px for a good calibration)
```









## 参考内容

1. https://blog.csdn.net/qq_38364548/article/details/124917067
2. http://zhaoxuhui.top/blog/2020/09/09/kalibr-installation-and-use.html
3. http://zhaoxuhui.top/blog/2020/09/29/intel-realsense-D435i-calibration-kalibr.html
4. https://github.com/gaowenliang
5. https://github.com/ethz-asl/kalibr/wiki/calibrating-the-vi-sensor
6. https://www.youtube.com/watch?app=desktop&v=puNXsnrYWTY

