# Unitree Go2 动捕闭环极简操作手册（速查版）

> **核心原则**：
> 1. 所有 ROS 2 Foxy 终端严禁开启 Conda（确保命令行无 `(base)`，先执行 `conda deactivate`）。
> 2. 路由器与交换机之间**只接单根网线**（优先保留 `2.5G` 口），严禁双线直连产生环路。（理论上确实这样，但实际不知道为什么没事）
> 3. 实机运行前留出直径 $2.0\text{ m}$ 以上空地，先用手柄让 Go2 站立，且**全程手持遥控器**随时接管。

## 一、三端网络信息速查

统一接入 5GHz Wi-Fi（`Turbot3-Multi`，`192.168.169.x` 网段）：

| 设备 | IP 地址 | 关键端口 / 接口 |
| --- | --- | --- |
| 网关路由器 | `192.168.169.1` | 5GHz AP |
| 动捕电脑（Windows） | `192.168.169.10` | VRPN `3883` |
| 控制主机（Ubuntu 20.04） | `192.168.169.16` | 无线网卡 `wlo1` |
| Go2 机载端（Ubuntu 20.04） | `192.168.169.17` | UDP `15001` |

---

## 二、动捕电脑配置（Windows `192.168.169.10`）

1. 打开 **XINGYING**，右下角加载 **`C:\jk_for_nokov`** 中的已有场地标定文件（无需重复挥杆标定），再点击连接镜头。
2. 将 Go2 机头朝向 $+X$ 方向放置，确认机身 4～5 颗非对称反光球识别稳定，已创建名为 **`go2`** 的刚体。
3. 确认处于 **实时模式（Live Mode）**，在 **Broadcast / VRPN Settings** 中开启 **VRPN** 广播（端口 `3883`），并确保 Windows 防火墙放行。

---

## 三、首次部署配置（环境已配好可直接跳至第四节）

### 1. 控制主机安装与编译 `vrpn_client_ros`

```bash
conda deactivate
which python3                  # 确认为 /usr/bin/python3
sudo ufw disable               # 关闭防火墙

sudo apt update && sudo apt install -y \
  ros-foxy-vrpn ros-foxy-geometry-msgs ros-foxy-tf2 ros-foxy-tf2-ros \
  ros-foxy-ament-cmake ros-foxy-ament-lint-auto ros-foxy-ament-lint-common ros-foxy-rviz2

source /opt/ros/foxy/setup.bash
mkdir -p ~/mocap_ws/src && cd ~/mocap_ws/src
git clone https://github.com/NOKOV-MOCAP/vrpn_client_ros.git

cd ~/mocap_ws
colcon build --symlink-install

# 写入动捕电脑 IP
sed -i "s/server: .*/server: 192.168.169.10/" ~/mocap_ws/src/vrpn_client_ros/config/sample.params.yaml
```

### 2. 控制主机锁定 5GHz AP 的 BSSID（防后台漫游扫描）

```bash
CONN_NAME=$(nmcli -t -f NAME,DEVICE connection show --active | grep ':wlo1$' | cut -d: -f1)
CUR_BSSID=$(iw dev wlo1 link | awk '/Connected to/ {print $3}')
sudo nmcli connection modify "$CONN_NAME" 802-11-wireless.bssid "$CUR_BSSID"
sudo nmcli connection up "$CONN_NAME"
```

### 3. 确认两端脚本已就位

* **控制主机**：`~/mocap_ws/go2_mocap_udp_ctrl.py`（单点）、`~/mocap_ws/go2_mocap_circle_ctrl.py`（圆周）、`~/mocap_ws/go2_mocap_circle_pid_ctrl.py`（圆周 PID）。
* **Go2 机载端**：`~/zjjgo2/run.sh`、`~/zjjgo2/mocap_udp_bridge.py`。

---

## 四、日常实机启动顺序（4 步直接运行）

### 第 1 步：动捕电脑

启动 **XINGYING** → 加载 `C:\jk_for_nokov` 标定文件并连接镜头 → 确认刚体 `go2` 追踪正常且 **VRPN（3883）** 已开启。

### 第 2 步：Go2 机载端（挂保活 + 启 UDP 桥接）

```bash
ssh unitree@192.168.169.17    # 提示选择 ROS 版本时输入 1 (foxy)

# 挂载后台防休眠保活进程
pgrep -af "ping -i 0.2 192.168.169.1" || nohup ping -i 0.2 192.168.169.1 > /dev/null 2>&1 &

# 启动抗积压桥接
source /opt/ros/foxy/setup.bash
source ~/zjjgo2/run.sh
python3 ~/zjjgo2/mocap_udp_bridge.py
```

### 第 3 步：控制主机终端 1（关省电 + 启 VRPN 接收）

```bash
conda deactivate 2>/dev/null || true
sudo iw dev wlo1 set power_save off

source /opt/ros/foxy/setup.bash
source ~/mocap_ws/install/setup.bash
export CYCLONEDDS_URI='<CycloneDDS><Domain><General><Interfaces><NetworkInterface name="lo"/></Interfaces></General></Domain></CycloneDDS>'
ros2 launch vrpn_client_ros sample.launch.py
```

*(确认终端输出 `Creating new tracker go2`)*

### 第 4 步：控制主机终端 2（启动控制脚本）

```bash
conda deactivate 2>/dev/null || true
source /opt/ros/foxy/setup.bash
export CYCLONEDDS_URI='<CycloneDDS><Domain><General><Interfaces><NetworkInterface name="lo"/></Interfaces></General></Domain></CycloneDDS>'

# 按需三选一执行（结束按 Ctrl+C 自动发送 StopMove 停稳）：
python3 ~/mocap_ws/go2_mocap_udp_ctrl.py           # 单点验证（前移 0.3 m）
python3 ~/mocap_ws/go2_mocap_circle_ctrl.py        # 圆周追踪（R=0.90m, vx=0.38m/s）
python3 ~/mocap_ws/go2_mocap_circle_pid_ctrl.py    # 圆周追踪 PID 版
```

---

## 五、极简自检命令（异常时核对）

```bash
# 1. 主机测三端链路与无线抖动（要求 0% 丢包、延迟平稳无上千毫秒尖峰）
ping -c 25 -i 0.2 192.168.169.10
ping -c 25 -i 0.2 192.168.169.17

# 2. 检查主机 wlo1 省电已关闭（要求输出 Power save: off）
iw dev wlo1 get power_save

# 3. 核对动捕话题频率（需先 export 同样的 CYCLONEDDS_URI，正常约 100 Hz）
ros2 topic hz /go2/pose
ros2 topic echo /go2/pose --once
```

*(Windows 动捕端反向测通命令：`ping -n 25 192.168.169.16`)*