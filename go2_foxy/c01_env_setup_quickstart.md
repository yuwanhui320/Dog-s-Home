# Unitree Go2 基础环境配置与网线直连验证指南（精简版）

- **文件名**：`c01_env_setup_quickstart.md`
- **适用平台**：**Unitree Go2** 四足机器人
- **适用系统**：**Ubuntu 20.04 LTS** + **ROS 2 Foxy**
- **文档定位**：仅保留从零配置到实机话题验证的必要执行命令与红线避坑，完整原理与图文说明见同目录 `c01_env_setup_guide.md`
- **操作习惯提示**：修改系统网络、依赖或脚本配置时建议同步记录，便于团队复现与排查
- **其他**：本教程依照bilibili宇树support所发布的话题，如果后面cyclonedds相关那部分编译失败，请参考G1教程：https://support.unitree.com/home/zh/G1_developer/ros2_communication_routine

# 目录

- [核心参数与红线](#核心参数与红线)
- [安装ROS2与基础依赖](#安装ros2与基础依赖)
- [下载与编译unitree_ros2](#下载与编译unitree_ros2)
- [网线直连配置与话题验证](#网线直连配置与话题验证)
- [高频报错速查](#高频报错速查)

# 核心参数与红线

本节汇总网线直连默认 IP 分配与编译铁律，执行命令前务必先核对。

>注意：本文档所有命令均基于 **Ubuntu 20.04 (ROS 2 Foxy)** 编写，其他系统版本（如 Ubuntu 22.04 Humble）请勿直接照搬。

## 1 直连 IP 对照表

| 设备/项目 | IP 地址 | 说明 |
| --- | --- | --- |
| 运控计算单元 | `192.168.123.161` | 机器狗底层运控主机（`ping` 测试目标） |
| NX 用户开发单元（拓展坞） | `192.168.123.18` | 机载电脑，SSH 账号 `unitree`，密码 `123` |
| 激光雷达 / M8 航插 | `192.168.123.20` | 选配雷达默认地址 |
| 开发主机（Ubuntu 20.04） | `192.168.123.99` | 手动静态 IP（避开 `.161` / `.18` / `.20`） |
| 子网掩码 | `255.255.255.0` | 掩码长度 `/24`，默认网关留空 |

## 2 环境隔离红线

>注意：安装 ROS 2 依赖、编译 **cyclonedds** 与 **unitree_ros2** 以及运行 ROS 2 命令行工具时，**严禁开启 Conda 或任何 Python 虚拟环境**，必须使用系统默认 `/usr/bin/python3`（Python 3.8）。

>注意：首次编译 `cyclonedds` 前，**严禁提前 `source /opt/ros/foxy/setup.bash`**；若 `~/.bashrc` 中已写入该行，务必先注释并重开终端再编译。

# 安装ROS2与基础依赖

完成基础工具、**ROS 2 Foxy** 与 **CycloneDDS** 相关系统依赖安装。

## 1 退出虚拟环境并安装基础工具

- 确认退出 Conda 并核对系统 Python 路径：

```bash
conda deactivate                 # 退出 Conda 环境（未安装 Conda 可跳过）
which python3                    # 预期输出：/usr/bin/python3
python3 --version                # 预期输出：Python 3.8.x
```

- 更新软件源并安装基础编译与网络工具：

```bash
sudo apt update
sudo apt install -y curl gnupg2 lsb-release git build-essential cmake net-tools locales
```

## 2 安装 ROS 2 Foxy 与 CycloneDDS 依赖

- 配置 UTF-8 编码并添加 ROS 2 官方软件源：

```bash
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

sudo apt install -y software-properties-common
sudo add-apt-repository universe
sudo curl -sSL [https://raw.githubusercontent.com/ros/rosdistro/master/ros.key](https://raw.githubusercontent.com/ros/rosdistro/master/ros.key) -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] [http://packages.ros.org/ros2/ubuntu](http://packages.ros.org/ros2/ubuntu) $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

- 安装 ROS 2 Foxy 桌面版及编译依赖：

```bash
sudo apt update
sudo apt install -y ros-foxy-desktop ros-foxy-rmw-cyclonedds-cpp ros-foxy-rosidl-generator-dds-idl python3-colcon-common-extensions
```

*备注1：若官方源下载缓慢，可使用鱼香 ROS 一键安装脚本完成 ROS 2 Foxy 安装，但仍需补装 `ros-foxy-rmw-cyclonedds-cpp` 与 `ros-foxy-rosidl-generator-dds-idl`。*

- 本机快速验证（任选两终端分别运行，收到消息后按 `Ctrl+C` 退出）：

```bash
source /opt/ros/foxy/setup.bash
ros2 run demo_nodes_cpp talker
```

```bash
source /opt/ros/foxy/setup.bash
ros2 run demo_nodes_py listener
```

# 下载与编译unitree_ros2

拉取 **unitree_ros2** 与适配 Go2 通信协议的 **CycloneDDS 0.10.x**，分两阶段完成编译。
详细操作参考：

```text
[https://github.com/unitreerobotics/unitree_ros2](https://github.com/unitreerobotics/unitree_ros2)
```

## 1 拉取仓库与 CycloneDDS 源码

- 克隆 `unitree_ros2`、`rmw_cyclonedds`（`foxy` 分支）与 `cyclonedds`（`releases/0.10.x` 分支）：

```bash
cd ~
git clone [https://github.com/unitreerobotics/unitree_ros2.git](https://github.com/unitreerobotics/unitree_ros2.git)
cd ~/unitree_ros2/cyclonedds_ws/src
git clone [https://github.com/ros2/rmw_cyclonedds](https://github.com/ros2/rmw_cyclonedds) -b foxy
git clone [https://github.com/eclipse-cyclonedds/cyclonedds](https://github.com/eclipse-cyclonedds/cyclonedds) -b releases/0.10.x
```

## 2 两阶段编译（严格按顺序）

>注意：开始前必须**新开一个干净终端**，确认未处于 Conda 环境且未加载任何 ROS 2 环境变量。

- 第一阶段：不 `source` ROS 2，单独编译 `cyclonedds`：

```bash
conda deactivate                 # 确保无虚拟环境
env | grep -i ros                # 预期输出为空；若有输出请先检查 ~/.bashrc 并重开终端
cd ~/unitree_ros2/cyclonedds_ws
colcon build --packages-select cyclonedds
```

- 第二阶段：激活 ROS 2 Foxy 后全量编译 `cyclonedds_ws`：

```bash
source /opt/ros/foxy/setup.bash
cd ~/unitree_ros2/cyclonedds_ws
colcon build
```

- 编译成功会看见以下终端输出：

```text
Summary: X packages finished [...]
```

# 网线直连配置与话题验证

配置开发主机有线网卡静态 IP，绑定 `setup.sh` 网卡名并验证读取 Go2 实机话题。

## 1 配置静态 IP 与连通性检查

- 网线直连开发主机与 Go2 机身以太网口，查看有线网卡名称：

```bash
ifconfig                         # 记录连接 Go2 的有线网卡名，如 enp3s0 / eth0
```

- 在 Ubuntu **设置 -> 网络 -> 有线 -> IPv4** 中选择 **手动（Manual）**，填入：
  - 地址：`192.168.123.99`
  - 子网掩码：`255.255.255.0`
  - 网关：留空

- 测试与 Go2 运控主机的连通性：

```bash
ping -c 4 192.168.123.161
```

- 连通正常会看见连续回包（`0% packet loss`）。

## 2 修改 setup.sh 并验证实机话题

- 编辑 `~/unitree_ros2/setup.sh`，将网卡名替换为本机实际有线网卡名：

```bash
vim ~/unitree_ros2/setup.sh
```

```bash
#!/bin/bash
echo "Setup unitree ros2 environment"
source /opt/ros/foxy/setup.bash
source $HOME/unitree_ros2/cyclonedds_ws/install/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI='<CycloneDDS><Domain><General><Interfaces>
                            <NetworkInterface name="<你的有线网卡名">" priority="default" multicast="default" />
                        </Interfaces></General></Domain></CycloneDDS>'
```

*备注1：Vim 中按 `i` 进入编辑模式，修改完成后按 `Esc`，输入 `:wq` 回车保存退出。*

- 激活环境并列出实机 ROS 2 话题：

```bash
source ~/unitree_ros2/setup.sh
ros2 topic list
```

- 通信正常会看见以下典型实机话题：

```text
/api/sport/request
/lowstate
/sportmodestate
/utlidar/cloud
/wirelesscontroller
```

- 抽检单帧高层状态数据，确认数据流正常刷新：

```bash
ros2 topic echo /sportmodestate --once
```

# 高频报错速查

| 现象/报错 | 原因 | 快速处理 |
| --- | --- | --- |
| `colcon build` 报 `empy` / `catkin_pkg` 缺失 | 处于 Conda 虚拟环境 | 执行 `conda deactivate`，删除 `build/ install/ log/` 后重编 |
| 编译 `cyclonedds` 失败或版本冲突 | 提前 `source` 了系统 ROS 2 | 注释 `~/.bashrc` 中的 Foxy `source` 行，新开终端删除构建目录重编 |
| `ping` 通但 `ros2 topic list` 无实机话题 | 网卡名写错、防火墙拦截或 `ROS_DOMAIN_ID` 不一致 | 核对 `setup.sh` 网卡名，执行 `sudo ufw disable`，确认 `echo $ROS_DOMAIN_ID` 为空或 `0` |
| 新开终端找不到 `unitree_go` 消息类型 | 未激活工作空间脚本 | 每次新开终端执行 `source ~/unitree_ros2/setup.sh` |