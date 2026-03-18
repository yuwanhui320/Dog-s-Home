# CPS Ubuntu电脑账户密码
双系统开机按F11选择win还是ubuntu
密码：win：Wang20240913；ubuntu：123456
（下载什么东西或者更改什么设置最好做个文档记录）

# 目录
- [机器狗操作](#机器狗操作)
- [机器狗仿真](#机器狗仿真)
- [机器狗与动捕系统连接与代码运行](#机器狗与动捕系统连接与代码运行)
- [用RVIZ画机器狗的运动轨迹](#用rviz画机器狗的运动轨迹)
- [在RVIZ里导入机器狗模型]（#在RVIZ里导入机器狗模型）

# 机器狗操作
通过ssh远程连接到机器狗，通过SDK控制机器狗关节运动
详细操作参考：
```
https://github.com/DeepRoboticsLab/Lite3_MotionSDK?tab=readme-ov-file#1-sdk-change-log
```
## 1 SDK下载
将**Lite3_MotionSDK**存储库克隆到本地主机：
	``` cd xxxxxxxxxx    #cd <to where you want to store this project>
 git clone --recurse-submodules https://github.com/DeepRoboticsLab/Lite3_MotionSDK.git
	```
## 2 识别远程主机地址、用户名和密码

### 2.1 识别运动主机地址
- 通过 WiFi 连接到机器人时，请验证您的开发主机的 IP 地址：
- 如果网段为 1，则运动主机 IP 地址为 192.168.1.120。
- 如果网段为 2，则运动主机 IP 地址为 192.168.2.1。
通过以太网端口连接到机器人时，运动主机IP地址为192.168.1.120。

### 2.2 识别用户名和密码
| 用户名 | 密码 |
| ysc    | , (单引号)|

## 3 配置运动主机
通过ssh远程连接到运动主机，为运动主机配置目标 IP 地址，以发送关节数据等运动数据。
- 首先连接机器狗wifi或以太网端口将开发主机连接到机器狗局域网，然后使用SSH连接上一步确定的地址和用户名的运动主机：
```
ssh ysc@192.168.1.120
```
- 连接主机后，在终端中输入以下命令，打开网络配置文件：
```
 cd ~/jy_exe/conf
 vim network.toml
 ```
 - 配置文件 network.toml 如下所示:
 ```
  ip = '192.168.1.102'  # Motion host will send data to this IP address
 target_port = 43897
 local_port = 43893
 ```
 *备注1：将ip地址改为自己的Ubuntu系统IP，可在设置中查看；多台电脑可考虑将ip设置为静态ip*
 *备注2：在更改ip时，点击“i”或“a”进行修改，点击esc退出编辑模式，输入：wq保存退出*
 
 - 重新启动运动程序以应用新配置：
```
 cd ~/jy_exe/scripts
 sudo ./stop.sh
 sudo ./restart.sh
 ```
 ## 4 配置MotionSDK
 main.cpp代码的第 39 行为 SDK 创建了一个发送线程，用于向机器人运动主机发送关节控制命令：
 ```
 Sender* send_cmd = new Sender("192.168.1.120",43893); ///< Create send thread
 ```
 请根据第 4 章中确定的运动主机地址修改目标 IP 地址。Sender()

 ## 5 编译并运行
 main.cpp提供了一个简单的站立演示，站立一段时间后，它将控制权直接返回给底层控制器，机器人自动进入阻尼保护模式。
**但为了保证SDK的安全使用，在main.cpp的原始代码中，注释掉了第73行发送关节控制命令的代码，所以机器人默认只会重置为零，不会站立：**
```
//send_cmd->SendCmd(robot_joint_cmd);
```
>谨慎：在取消注释之前，开发者必须确保 SDK 与机器人之间的通信正常运行（参见“5.1 检查通信”），并确保 SDK 发送的联合控制命令正确无误，即向开发主机返回正确的关节信息

### 5.1 检查通信
MotionSDK 使用 UDP 与机器人进行通信。

要检查机器人是否成功向 SDK 发送数据，开发者可以使用 SDK 打印关节数据或 imu 数据等数据，以判断 SDK 是否接收了机器人发送的数据，或者在运行 Demo 时观察是否打印。No data from the robot was received!!!!!!

要检查SDK是否成功向机器人发送控制命令，开发人员可以在运行演示时观察机器人的动作。如果机器人可以归零，则证明SDK可以成功向机器人发送命令。
- 首先编译原始代码
>注意：您可以在编译前取消注释 main.cpp 中的第 75 行，以使 SDK 打印 imu 数据，以判断 SDK 是否接收到机器人发送的数据。cout << robot_data->imu.acc_x << endl
- 进入解压缩的文件夹，在与CMakeLists.txt相同的目录下创建一个新的构建目录;
```
 cd xxxxxxxx     # cd <path to where you want to create build directory>
 mkdir build
 ```
 >注意：开发人员可以在任何地方创建构建目录。但是，运行 CMake 时需要CMakeLists.txt路径。
 - 导航到构建目录，然后编译：
	- 为 x86 主机编译：
	```
	  cd build
      cmake .. -DBUILD_PLATFORM=x86     # cmake <path to where the CMakeLists.txt is>
      make -j
   ```
	>针对ARM的主机编译请参考原GitHub链接

- 编译完成后，build目录下会生成一个名为**Lite_motion**的可执行文件，这是编译的结果;
- 在终端中输入以下命令以运行Lite_motion（运行前请确保开发主机已连接到机器人网络）：
```
 ./Lite_motion
 ```
 >注意：为了使机器人顺利归零，请在运行程序之前将机器人调整到就绪位置。
 - 观察机器人在运行Lite_motion时是否复位归零，在终端中打印机器人发送的数据是否正常(要输出机器狗关节信息)

### 5.2 通信故障排除
如果运动主机用户名为ysc，则 MotionSDK 正在开发主机上运行，并且开发主机通过 WiFi 连接到机器人，如果SDK收到的信息均为0，则在ysc@ysc:~/jy_exe/conf文件中输入：
```
sudo tcpdump -x port 43897 -i p2p0
```
>其余情况请查阅原GitHub文档

### 5.3 编译与开发
在确保 SDK 与机器人正确通信并且您的控制命令正确后，您可以在 main.cpp 中取消注释第 73 行中的代码，重新编译并再次运行它：send_cmd->SendCmd(robot_joint_cmd)

- 删除之前生成的构建目录
- 打开一个新的终端，创建一个空的构建目录;
```
 cd xxxxxxxx     # cd <path to where you want to create build directory>
 mkdir build
 ```
- 导航到构建目录，然后编译;
  - 为 x86 主机编译：
	```
	 cd build
      cmake .. -DBUILD_PLATFORM=x86     # cmake <path to where the CMakeLists.txt is>
      make -j 
    ```
-编译后，在构建目录中生成一个名为 Lite_motion 的可执行文件。在终端中输入以下代码以运行程序：
```
 ./Lite_motion
 ```
 >注意：使用绝影Lite3测试运动控制算法或进行实验时，所有在场人员应与机器人保持至少5米的距离，并将机器人悬挂在机器人吊装装置上，以免对人员和设备造成意外损坏。如果用户在实验过程中需要接近机器人，用户必须确保机器人进入紧急停止状态或使用命令关闭运动程序。sudo ./stop.sh

# 机器狗仿真

## 1 仿真环境部署
### 1.1 在linux系统上安装anaconda3
- 安装**anaconda**（参考）：
  ```
  https://blog.csdn.net/wyf2017/article/details/118676765 
  ```
- 安装虚拟环境
  ```
  conda create-n dr_gym python=3.8
  ```
- 确认环境是否已经正确安装
  ```
  conda env list
  ```
  ```
  #conda environnents
   
  base    */home/ysc/anaconda3
  dr_gym  /home/ysc/anaconda3/envs/dr_gym
  ```
-  后续所有的操作都是在**dr_gym** 环境中
  ```
  conda activate dr_gym
  ```

### 1.2  Ubuntu20.04 安装 Isaac Gym 仿真器：
- 安装**Isaac Gym**仿真器教程：
```
https://blog.csdn.net/qq_54900679/article/details/147387701
```
>注意: *根据教程，需先安装显卡驱动与CUDA，CUDA版本需与显卡匹配，cps安装的cuda版本为12.8.1*
- Ubuntu20.04安装**Nvidia显卡**驱动教程：
```
https://blog.csdn.net/ytusdc/article/details/132403852 
```
- Ubuntu 20.04 **CUDA**安装：
```
参考1 https://cloud.tencent.com/developer/article/2037551  
参考2 https://blog.csdn.net/CC977/article/details/122789394
```
- 最后安装python包及相关依赖**运行超时**问题解决办法参考:
```
pip install -e .
```
  
```
https://blog.csdn.net/qlkaicx/article/details/146703877
```
### 1.3 安装其他依赖项
- 安装**rs1_rl**
  ```
  git clone https://github.com/DeepRoboticsLab/Robot_Training_Cases/tree/main/Case2
  ```
- 在给出的代码中的rsl_r1目录下运行
  ```
    pip-e
  ```
- 安装其他需要安装的库
	```
	pip install setuptools==59.5.0
	pip3 install torch==2.3.1+cu121 torchvision==0.18.1+cu121 torchaudio==2.3.1+cu121 --index-url https://download.pytorch.org/whl/cu121
	```
  >注意:*需选择与显卡适配的**pytorch**版本,已安装的**pytorch**版本为**2.3.1***
	验证**pytorch**的安装情况
	```
	python3 -c "import torch; print(torch.cuda.is_available())"
	```
###  1.4 运行dr_gym/legged gym/scripts/train.py脚本，观察是否运行正常
```
conda activate isaac
cd dr_gym/legged gym/scripts
python3 train.py
```
###  1.4 运行dr_gym/legged gym/scripts/train.py脚本，观察是否运行正常
```
conda activate isaac
cd dr_gym/legged gym/scripts
python3 train.py
```


# 2 运行案例6：基于VMC的运动控制
## 2.1 下载代码包

    git clone https://github.com/DeepRoboticsLab/Lite3_VMC.git
## 2.2 编译
需在系统全局环境下进行编译与仿真。
- 编译

>若终端为（base）ubuntu20@ubuntu20:
>则运行conda deactivate
```
    cd ~/Lite3_VMC
    catkin_make
```
>注意：这里可能会由于缺少某些依赖项而编译失败，需安装对应的依赖项，再进行编译。再次编译时，需先将工作空间中已生成的build和include文件夹删除，再进行编译。
>
>若使用catkin_make进行编译出现anaconda路径污染问题，借助chatgpt对相关报错进行处理。

- 配置环境变量
```
   source ~/Lite3_VMC/devel/setup.bash
```
- 运行launch文件，加载gazebo仿真环境，加载机器狗仿真模型
```
    roslaunch gazebo_model_spawn gazebo_startup.launch
    roslaunch gazebo_model_spawn model_spawn.launch rname:=lite3 use_xacro
=true use camera:=false #start controller
```
- 按enter键
- 运行Lite3主控程序，键盘控制程序，通过键盘对机器狗仿真模型进行运动控制
```
    rosrun examples example_lite3_sim
    rosrun examples example_keyboard
```
>"l"：切换到“力矩支撑模式”；
>"j"：切换步态；
>"u"：站立/坐下；
>"i"：坐下后退出程序；
>"w/s/a/d"：前后左右移动；
>"q/e"：向左/右旋转；
>"ctrl+C"：退出键盘控制程序

#  机器狗与动捕系统连接与代码运行

### 1 确保动捕系统主机连接机器狗的WIFI
- 名称：YCS-JYML-td9lsi  密码：12345678
### 2 在动捕系统XINGYING APP中标定机器狗
- C:\jk_for_novkov路径下查看是否已有dog.mars标定文件，已有可直接打开软件，点击开始即可
- 若需重新标定，到以上标定文件路径中将已有标定文件删去，打开软件，点击冻结帧，按住shift选中要标定的对象，右键点击create rig，命名即可
### 3 控制主机操作
-  连接机器狗WIFI
-  测试 Ubuntu 是否跟 XINGYING 软件所在主机的网络连通，两台电脑的 IP 地址一定要在同一个网段（故均连接机器狗WIFI）。此处的IP为 XINGYING 软件所在主机的IP。若 PING 不通，请检查电脑的 IP 地址是否设置正确。**[插入图片]**
  ``` bash
  ping 192.168.2.65
  ```  
  -  ping通后，启动动捕系统的ros接口，此处的IP为 XINGYING 软件所在主机的IP。
  ``` bash
  roslaunch vrpn_client_ros sample.launch server:=192.168.2.65
  ```  
### 4 运行控制代码
- 在VS Code中或终端中运行均可。遇到紧急情况Ctrl+C结束运行。


# 用RVIZ画机器狗的运动轨迹
呈现出的效果是，从上往下看视角看到的机器狗实时运动的画面和实时在动捕系统上画出的轨迹图
 
## 1 电脑WiFi连接
将开启**rivz**的那台电脑
连接上狗的WiFi **YSC—JYML—td9lsl—5G**
  
## 2 在Ubuntu里的操作步骤

### 2.1 启动摄像头
- 在桌面另开一个终端，输入
```
    roslaunch mindvision_cam ge300gc.launch
```

- 开启成功，会看见
```
    camera：0 open successed！！！
```

### 2.2 打开rviz
- rviz在Ubuntu的桌面开启终端，输入
```
    rviz
```

- 开启成功，会看见弹出相应界面

## 3 在rviz里的操作步骤

### 3.1 将设定好的配置添加进rviz
- 点击rviz界面左上角的**interact**
- 在弹出的界面点击**Open Config**
- 在弹出的界面左侧点击**src**
- 选择需要的文件（狗的圆形轨迹规划的相关文件保存在了2025XuKe中），点击文件2025XuKe，可以看见文件dog_rviz.rviz，点击文件dog_rviz.rviz，就可以看见摄像头监控的机器狗实时运动的画面和实时在动捕系统上画出的轨迹图

### 3.2 连接动捕系统，将数据传入rviz
 - 接着3.1操作的界面，点击左下角的**add**，确认**Path**和**TF**被勾选（点开**Path**里面的选项都没有被勾选）
 - 在Ubuntu的桌面开启一个新的终端，输入
 ```
    roslaunch vrpn_client_ros sample.launch server:=192.168.2.65
 ```  

  连接动捕系统，得到动捕系统的数据，连接成功会看见类似
  ```
      Found new sender:dog    
      Creating new tracker dog
  ```
 - 回到rviz界面，可以看到狗的坐标（如果不需要原点坐标，可以在**interact**里面把**world**不勾选，原点坐标就会消失）

 
## 4 屏幕录制
- 开启Kazam屏幕录制
- 在右上角结束录制，录制的文件在ubuntu的视频里面  


# 在RVIZ里导入机器狗模型
注意：固态硬盘里的ros版本为22.04（ros2 humble）对于18.04版本，不可以直接参考

## 1 创建并初始化 ROS 2 工作空间

### 1.1 创建工作空间目录
- 命名为 ros2_ws
  ```
     mkdir -p ~/ros2_ws/src
     cd ~/ros2_ws/src
  ```

### 1.2 创建工作空间目录
- 包含 URDF 加载所需所有依赖
  ```
     ros2 pkg create --build-type ament_python robot_urdf \
  --dependencies rclpy urdf robot_state_publisher joint_state_publisher_gui rviz2
  ```

### 1.3 编译工作空间
- --symlink-install 避免重复编译
  ```
    cd ~/ros2_ws
    colcon build --symlink-install
  ```
>注意：系统里安装的是 ROS 2 Humble，ROS 2 已经抛弃了catkin，改用 colcon 作为编译工具。


### 1.4 激活工作空间
- 必须执行，否则找不到自定义包
  ```
   source install/setup.bash
  ```
        
## 2 存放 URDF 文件

### 2.1 创建 URDF 文件夹并放入模型文件
- 将下载的 Lite3 的 URDF 文件Lite3_high_res.urdf复制到该文件夹
  ```
   cd ~/ros2_ws/src/robot_urdf
   mkdir urdf
  ```

### 2.2 编写 ROS 2 启动文件
- 创建 launch 文件夹并新建文件
  ```
   cd ~/ros2_ws/src/robot_urdf
   mkdir launch
   touch launch/display_robot.launch.py
  ```
- 在当前终端输入以下命令，打开 Ubuntu 自带的文本编辑器（Gedit）
  ```
   gedit display_robot.launch.py
  ```

- 打开 display_robot.launch.py，复制以下内容
  ```
   from launch import LaunchDescription
   from launch_ros.actions import Node
   from launch.substitutions import Command, FindExecutable, PathJoinSubstitution
   from launch_ros.substitutions import FindPackageShare

   def generate_launch_description():
    # 1. 配置 URDF 文件路径
    urdf_file_name = "Lite3_high_res.urdf"
    urdf_path = PathJoinSubstitution([
        FindPackageShare("robot_urdf"),
        "urdf",
        urdf_file_name
    ])

    # 2. 加载 URDF 内容到参数服务器
    robot_description = Command([FindExecutable(name="xacro"), " ", urdf_path])

    # 3. 启动机器人状态发布节点（核心）
    robot_state_publisher = Node(
        package="robot_state_publisher",
        executable="robot_state_publisher",
        parameters=[{"robot_description": robot_description}]
    )

    # 4. 启动关节状态发布节点（带 GUI 调节面板）
    joint_state_publisher_gui = Node(
        package="joint_state_publisher_gui",
        executable="joint_state_publisher_gui",
        name="joint_state_publisher_gui"
    )

    # 5. 启动 RViz2 可视化工具
    rviz2 = Node(
        package="rviz2",
        executable="rviz2",
        name="rviz2",
        output="screen"
    )

    # 组装并返回启动描述
    return LaunchDescription([
        robot_state_publisher,
        joint_state_publisher_gui,
        rviz2
    ])

  ```
> 注意：ROS 2 的 Python 功能包（ament_python 类型）不会自动把 launch/urdf/config 等自定义文件夹复制到 install 目录，必须在 setup.py 中明确声明这些文件的安装路径，否则 ros2 launch 会找不到文件（即使文件在 src 目录下）。

>解决方法：
>#### 1 定位并编辑 setup.py 文件
>cd ~/ros2_ws/src/robot_urdf(切换到 robot_urdf 包目录)
>gedit setup.py(用 gedit 打开 setup.py)
>
>#### 2 修改 setup.py，添加文件安装规则
>打开后你会看到默认的 setup.py 内容，需要做两处修改：
>在 from setuptools import setup 下面添加一行：
>import os
>from glob import glob
>from setuptools import setup
>
>修改 data_files 参数:
>找到文件中 data_files 这一段（默认是空的 data_files=[],），替换成以下内容
>data_files=[
    # 1. 把 package.xml 安装到共享目录（默认已有，保留）
    ('share/ament_index/resource_index/packages',
        ['resource/' + 'robot_urdf']),
    ('share/' + 'robot_urdf', ['package.xml']),
    # 2. 新增：安装 launch 文件夹到共享目录
    (os.path.join('share', 'robot_urdf', 'launch'), glob('launch/*.launch.py')),
    # 3. 新增：安装 urdf 文件夹到共享目录（后续加载 URDF 也需要）
    (os.path.join('share', 'robot_urdf', 'urdf'), glob('urdf/*.urdf')),
],
>
>#### 3 重新编译并激活环境
>修改 setup.py 后，必须重新编译才能让规则生效
>cd ~/ros2_ws(回到工作空间根目录)
>colcon build --symlink-install --packages-select robot_urdf(重新编译 --symlink-install 确保文件同步)
>source install/setup.bash(重新激活环境)
>
>#### 4 验证并重新启动 launch 文件
>ros2 launch robot_urdf display_robot.launch.py(再次执行 launch 命令)
>
>以上参考方法尝试之后还是报错，可以问ai解决方法

## 3 编译并启动 URDF 加载程序

### 3.1 重新编译
- 修改 launch 文件后必做
  ```
   cd ~/ros2_ws
   colcon build --symlink-install
   source install/setup.base
  ```

### 3.2 启动 launch 文件
- 正常现象：会弹出两个窗口 ——RViz2（空白）、关节调节面板（Joint State Publisher GUI）
  ```
   ros2 launch robot_urdf display_robot.launch.py
  ```

>某些工具包不属于Humble 核心安装包（默认不会自动安装），需要手动通过 apt 安装，否则运行会报错
>
>执行以下命令安装缺失的依赖包
>sudo apt update
>sudo apt install ros-humble-joint-state-publisher-gui(安装 ROS 2 Humble 版本的 joint_state_publisher_gui)

## 4 配置 RViz2 显示机器人模型

### 4.1 打开 RViz2 的 Displays 面板
- RViz2 窗口左侧有一个 Displays 面板（如果没看到，点击窗口左侧的「Displays」标签，或按 Ctrl+D 调出）
- 面板顶部会显示 Global Options（全局选项），下方是已加载的显示项（初始为空）

### 4.2 添加 RobotModel 显示项
- 在 Displays 面板底部，找到 Add 按钮，点击它
- 弹出「Add Display」窗口，左侧列表向下滚动，找到 RobotModel
- 选中 RobotModel 后，点击窗口右下角的 OK 按钮
此时 Displays 面板中会新增 RobotModel 选项（默认是红色叉号，因为参数未配置）

### 4.3 配置 RobotModel 核心参数
- 点击 Description Source 右侧下拉框，选择 Topic
- 在 Description Topic 输入框中，输入：/robot_description
- 在左侧 Global Options → Fixed Frame 输入框中，输入你的 URDF 根连杆名称（常见值）：Torso

>如果不确定需要在终端查询URDF 文件的根连杆名称

### 4.4 验证是否生效
- 配置完成后，RViz2 中 RobotModel 前面的红色叉号会变成绿色对勾，3D 视图区会显示机器人模型
- 若仍不显示，执行以下命令检查话题是否正常。正常会输出一大段 URDF 的 XML 内容，说明话题数据正常
  ```
   ros2 topic echo /robot_description
  ```

### 4.5 保存配置，避免下次重复设置
- 先创建 config 文件夹（存放配置文件）
  ```
   cd ~/ros2_ws/src/robot_urdf
   mkdir -p config 
  ```
- 在 RViz2 窗口中，点击左上角的 File → 选择 Save Config As
- 在弹出的文件保存窗口中，导航到 ~/ros2_ws/src/robot_urdf/config 目录
- 文件名输入 robot_view.rviz，点击 Save 保存

>模型下载链接：https://github.com/DeepRoboticsLab/deep_robotics_model

  

