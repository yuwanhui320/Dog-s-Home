# Unitree Go2 无线键盘控制与 ROS 2 话题桥接操作手册

本文档记录 Ubuntu 20.04 主机（`cairen@foxy2004`）通过 Wi-Fi 热点无线控制 Unitree Go2 机载电脑（`unitree@10.42.0.211`）的完整实验流程、网卡绑定配置、源码部署与安全验证方法。
（下载新工具、修改网络配置或更新脚本路径后建议同步更新本文档）

# 目录

- [架构与环境参数确认](#架构与环境参数确认)
- [主机热点开启与网卡绑定](#主机热点开启与网卡绑定)
- [核心脚本编写与部署](#核心脚本编写与部署)
- [空跑验包与话题核查](#空跑验包与话题核查)
- [实机底层转发与安全测试](#实机底层转发与安全测试)
- [常见故障排除与动捕扩展](#常见故障排除与动捕扩展)

# 架构与环境参数确认

本方案采用 **主机 ROS 2 话题发布 -> 主机 UDP 发包 -> 机载电脑单网卡转发底层** 的混合桥接架构，既在主机侧保留标准 ROS 2 话题生态（便于后续无缝接入动捕 ROS 话题），又避免 ROS 2 Foxy 的 CycloneDDS 跨无线热点网段与底层网段直接做节点发现引发的冲突与丢包。

注意：以下操作基于 **Ubuntu 20.04 (ROS 2 Foxy)** 与 **Unitree Go2** 机载环境，运行前请确保主机终端已退出 Conda 环境（如有 `(base)` 前缀需先执行 `conda deactivate`）。

## 1 通信链路说明

整个控制链路分为三段接力执行：

- **主机端 ROS 2 内部**：`~/go2_key_ros_pub.py` 读取键盘输入并发布 `/pc/key_request`（消息类型 `unitree_api/msg/Request`），同机运行的 `~/pc_ros_to_udp.py` 订阅该话题。
- **主机至机载电脑无线链路**：`~/pc_ros_to_udp.py` 将 `api_id` 与 `parameter` 打包为 JSON，通过 UDP 点对点发往 Go2 机载电脑的 `15001` 端口，不走跨网段 DDS 发现。
- **机载电脑至底层运控**：Go2 机载电脑在单网卡环境下运行 `~/zjjgo2/udp_relay.py`，接收 `15001` 端口 UDP 数据并发布到底层标准运控话题 `/api/sport/request`。

## 2 网络与设备参数对照

| 设备/项目 | 网卡或接口 | 地址/参数 | 说明 |
| --- | --- | --- | --- |
| 开发主机（Ubuntu 20.04） | `wlo1`（无线网卡） | `10.42.0.1/24` | 发射热点 `Go2_Hotspot`，热点网段网关 |
| 开发主机（有线口） | `enp5s0` | 未连接（`unavailable`） | 无线控制时需防止 CycloneDDS 误绑定该网卡 |
| Go2 机载电脑 | 无线网卡 | `10.42.0.211` | 以实际 `ip neigh` 查询结果为准 |
| SSH 登录账号 | `unitree@10.42.0.211` | 登录提示选 `1`（ROS 2 Foxy） | 实机调试期间建议仅保持 1 个 SSH 窗口 |
| UDP 中转端口 | 机载监听 `0.0.0.0` | `15001` | 承载主机至机载电脑的 JSON 控制指令 |
| 主机侧控制话题 | ROS 2 本地话题 | `/pc/key_request` | `unitree_api/msg/Request`，仅主机侧可见 |
| 底层运控话题 | 机载单网卡 ROS 2 | `/api/sport/request` | Go2 底层运控节点实际订阅的标准话题 |

# 主机热点开启与网卡绑定

本模块完成 Ubuntu 主机无线热点建立、Go2 连通性验证以及主机侧 CycloneDDS 显式绑定无线网卡 `wlo1`。

## 1 开启主机 Wi-Fi 热点

### 1.1 查看网卡状态并重建热点

- 查看当前主机所有网卡名称与状态，确认无线网卡为 `wlo1`：

```bash
nmcli device status
```

- 预期看到 `wlo1` 的 `TYPE` 为 `wifi`。若此前存在绑定错误网卡（如 `enp5s0`）的旧 `Hotspot` 配置，先删除再用 `wlo1` 重建热点：

```bash
nmcli connection delete Hotspot 2>/dev/null || true
nmcli device wifi hotspot ifname wlo1 ssid "Go2_Hotspot" password "12345678"
```

*备注1：开启热点后 `wlo1` 会从原外部 Wi-Fi 切换为热点发射模式，原本依赖外网或代理（如 Mihomo）的连接会同步变化，请确认控制链路不依赖外部代理。*

### 1.2 确认热点 IP 与 Go2 连通性

- 查看主机 `wlo1` 分配到的网关地址：

```bash
ip a show wlo1
```

- 正常可在输出中看到 `10.42.0.1/24`。等待 Go2 连上 `Go2_Hotspot`（约 30 秒）后，查询 Go2 获取到的 IP 地址并测试连通性：

```bash
ip neigh | grep 10.42.0
ping -c 3 10.42.0.211    # 若查询到的 IP 不同，请替换为实际 IP
```

- 正常现象为 3 个 ICMP 包全部收到回复且丢包率为 `0%`。

## 2 主机端绑定 CycloneDDS 网卡

主机 `~/unitree_ros2/setup.sh` 默认可能将 `CYCLONEDDS_URI` 绑定至有线网卡 `enp5s0`，在纯无线热点模式下会导致 ROS 2 节点创建失败。通过独立 XML 文件显式绑定 `wlo1` 可避免命令行手写 XML 引号出错。

### 2.1 创建 `~/cyclone_wlo1.xml`

- 在 Ubuntu 主机终端生成 CycloneDDS 配置文件：

```bash
cat << 'EOF' > ~/cyclone_wlo1.xml
<?xml version="1.0" encoding="UTF-8" ?>
<CycloneDDS xmlns="[https://cdds.io/config](https://cdds.io/config)" xmlns:xsi="[http://www.w3.org/2001/XMLSchema-instance](http://www.w3.org/2001/XMLSchema-instance)" xsi:schemaLocation="[https://cdds.io/config](https://cdds.io/config) [https://raw.githubusercontent.com/eclipse-cyclonedds/cyclonedds/master/etc/cyclonedds.xsd](https://raw.githubusercontent.com/eclipse-cyclonedds/cyclonedds/master/etc/cyclonedds.xsd)">
  <Domain id="any">
    <General>
      <Interfaces>
        <NetworkInterface multicast="default" name="wlo1" priority="default"/>
      </Interfaces>
    </General>
  </Domain>
</CycloneDDS>
EOF
```

### 2.2 激活主机 ROS 2 环境与网卡配置

- 之后在每个需要运行 ROS 2 节点或 `ros2 topic` 命令的主机终端中，先执行以下三行命令：

```bash
source /opt/ros/foxy/setup.bash
source ~/unitree_ros2/setup.sh 2>/dev/null || true
export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"
```

*备注1：如需新开主机终端自动绑定 `wlo1`，可将 `export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"` 追加写入 `~/.bashrc`，但注意在 `source ~/unitree_ros2/setup.sh` 之后执行，防止被覆盖。*

# 核心脚本编写与部署

本模块在 Ubuntu 主机端创建键盘发布节点与 ROS->UDP 桥接器，并在 Go2 机载电脑端创建 UDP->底层转发脚本。Python 节点只要调用 `rclpy.init()` 即可直接通过 `python3` 运行，无需打包后用 `ros2 run` 启动。

## 1 主机端脚本编写

### 1.1 创建键盘 ROS 2 发布节点 `~/go2_key_ros_pub.py`

- 在 Ubuntu 主机终端生成键盘发布脚本，负责读取按键并发布 `/pc/key_request`：

```bash
cat << 'EOF' > ~/go2_key_ros_pub.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json
import sys
import termios
import threading
import tty
import rclpy
from rclpy.node import Node
from unitree_api.msg import Request

MOVE = 1008
BALANCE_STAND = 1002

MOVE_KEYS = {
    "q": (1, 0, 1),   "w": (1, 0, 0),   "e": (1, 0, -1),
    "a": (0, 0, 1),                     "d": (0, 0, -1),
    "z": (-1, 0, -1), "s": (-1, 0, 0),  "c": (-1, 0, 1),
    "Q": (1, 1, 0),   "W": (1, 0, 0),   "E": (1, -1, 0),
    "A": (0, 1, 0),                     "D": (0, -1, 0),
    "Z": (-1, 1, 0),  "S": (-1, 0, 0),  "C": (-1, -1, 0),
}

SPEED_KEYS = {
    "r": (1.1, 1.1), "t": (0.9, 0.9),
    "f": (1.1, 1.0), "g": (0.9, 1.0),
    "v": (1.0, 1.1), "b": (1.0, 0.9),
}

ACTION_KEYS = {
    "h": 1016,  # 打招呼
    "j": 1031,  # 向前跳（室内慎用）
    "k": 1017,  # 伸展
    "n": 1009,  # 坐下
    "m": 1010,  # 从坐姿站起
    "y": 1022,  # 舞蹈 1
    "u": 1023,  # 舞蹈 2
}


def get_key():
    fd = sys.stdin.fileno()
    old = termios.tcgetattr(fd)
    try:
        tty.setraw(fd)
        return sys.stdin.read(1)
    finally:
        termios.tcsetattr(fd, termios.TCSADRAIN, old)


def main():
    rclpy.init()
    node = Node("go2_key_teleop_node")
    pub = node.create_publisher(Request, "/pc/key_request", 10)
    threading.Thread(target=rclpy.spin, args=(node,), daemon=True).start()

    lin_speed = 0.3
    ang_speed = 0.5
    print("[ROS2 键盘节点已启动] 话题: /pc/key_request | g:降速 h:打招呼 w/a/s/d:移动 x/空格:站立 Ctrl+C:退出")

    try:
        while rclpy.ok():
            key = get_key()
            if key == "\x03":
                break

            if key in SPEED_KEYS:
                lin_speed = max(0.1, min(1.0, lin_speed * SPEED_KEYS[key][0]))
                ang_speed = max(0.1, min(1.5, ang_speed * SPEED_KEYS[key][1]))
                print(f"\r[速度] 线速度 {lin_speed:.2f} m/s, 角速度 {ang_speed:.2f} rad/s          ", end="", flush=True)
                continue

            if key in MOVE_KEYS:
                vx, vy, vz = MOVE_KEYS[key]
                api_id = MOVE
                param = json.dumps({
                    "x": round(vx * lin_speed, 3),
                    "y": round(vy * lin_speed, 3),
                    "z": round(vz * ang_speed, 3),
                })
            elif key in ACTION_KEYS:
                api_id = ACTION_KEYS[key]
                param = "{}"
            else:
                api_id = BALANCE_STAND
                param = "{}"

            msg = Request()
            msg.header.identity.api_id = api_id
            msg.parameter = param
            pub.publish(msg)
            print(f"\r[发布 /pc/key_request] api_id={api_id} param={param}          ", end="", flush=True)
    finally:
        stop_msg = Request()
        stop_msg.header.identity.api_id = BALANCE_STAND
        stop_msg.parameter = "{}"
        pub.publish(stop_msg)
        node.destroy_node()
        rclpy.shutdown()
        print("\n已安全退出。")


if __name__ == "__main__":
    main()
EOF
```

### 1.2 创建主机 ROS->UDP 桥接脚本 `~/pc_ros_to_udp.py`

- 在 Ubuntu 主机终端生成桥接脚本，订阅 `/pc/key_request` 并通过 UDP 发往 Go2 机载电脑：

```bash
cat << 'EOF' > ~/pc_ros_to_udp.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json
import socket
import rclpy
from rclpy.node import Node
from unitree_api.msg import Request

GO2_IP = "10.42.0.211"    # 若 Go2 热点 IP 变化，请同步修改此处
GO2_PORT = 15001


class RosToUdpBridge(Node):
    def __init__(self):
        super().__init__("pc_ros_to_udp_bridge")
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.create_subscription(Request, "/pc/key_request", self.cb, 10)
        self.get_logger().info(f"已订阅 /pc/key_request -> UDP 发送至 {GO2_IP}:{GO2_PORT}")

    def cb(self, msg: Request):
        pkt = json.dumps({
            "api_id": int(msg.header.identity.api_id),
            "parameter": str(msg.parameter),
        }).encode("utf-8")
        self.sock.sendto(pkt, (GO2_IP, GO2_PORT))

    def destroy_node(self):
        self.sock.close()
        super().destroy_node()


def main():
    rclpy.init()
    node = RosToUdpBridge()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == "__main__":
    main()
EOF
```

## 2 机载端转发脚本部署

### 2.1 创建机载 UDP->底层中转脚本 `~/zjjgo2/udp_relay.py`

- 通过唯一 SSH 窗口登录 Go2 机载电脑（登录出现 `ros:foxy(1) noetic(2) ?` 时输入 `1` 并回车）：

```bash
ssh unitree@10.42.0.211    # 替换为 Go2 实际热点 IP
```

- 创建 `~/zjjgo2` 目录并写入 `~/zjjgo2/udp_relay.py`：

```bash
mkdir -p ~/zjjgo2
cat << 'EOF' > ~/zjjgo2/udp_relay.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json
import socket
import rclpy
from rclpy.node import Node
from unitree_api.msg import Request

UDP_IP = "0.0.0.0"
UDP_PORT = 15001


def main():
    rclpy.init()
    node = Node("go2_udp_relay")
    pub = node.create_publisher(Request, "/api/sport/request", 10)

    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind((UDP_IP, UDP_PORT))
    sock.settimeout(0.5)

    print(f"UDP->底层已启动: 监听 {UDP_IP}:{UDP_PORT} -> /api/sport/request")

    try:
        while rclpy.ok():
            try:
                data, _ = sock.recvfrom(4096)
            except socket.timeout:
                continue

            try:
                p = json.loads(data.decode("utf-8"))
                msg = Request()
                msg.header.identity.api_id = int(p.get("api_id", 1002))
                msg.parameter = str(p.get("parameter", "{}"))
                pub.publish(msg)
                print(f"[传底层] api_id={msg.header.identity.api_id} param={msg.parameter}")
            except Exception as e:
                print(f"[解析失败] {e}")
    except KeyboardInterrupt:
        pass
    finally:
        sock.close()
        node.destroy_node()
        rclpy.shutdown()


if __name__ == "__main__":
    main()
EOF
```

# 空跑验包与话题核查

坚持 **先空跑验包、再上真机** 的安全原则。本阶段机载电脑仅监听 UDP 端口并打印原始 JSON，不向 `/api/sport/request` 发布任何运动指令，确保机器狗保持绝对静止。

## 1 机载端启动纯 UDP 空跑监听

- 在 Go2 机载电脑 SSH 窗口运行纯收包脚本：

```bash
python3 -c '
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.bind(("0.0.0.0", 15001))
print("空跑监听中（狗绝对不动）...")
while True:
    data, addr = s.recvfrom(4096)
    print(f"收到 {addr}: {data.decode()}")
'
```

- 启动成功终端会停留并显示：

```text
空跑监听中（狗绝对不动）...
```

## 2 主机端启动桥接器与键盘节点

- 打开 **主机终端 A**，加载环境并启动 ROS->UDP 桥接器：

```bash
source /opt/ros/foxy/setup.bash
source ~/unitree_ros2/setup.sh 2>/dev/null || true
export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"
python3 ~/pc_ros_to_udp.py
```

- 启动成功会看见以下终端输出：

```text
[INFO] [pc_ros_to_udp_bridge]: 已订阅 /pc/key_request -> UDP 发送至 10.42.0.211:15001
```

- 打开 **主机终端 B**，加载环境并启动键盘 ROS 2 发布节点：

```bash
source /opt/ros/foxy/setup.bash
source ~/unitree_ros2/setup.sh 2>/dev/null || true
export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"
python3 ~/go2_key_ros_pub.py
```

- 启动成功会看见以下终端输出：

```text
[ROS2 键盘节点已启动] 话题: /pc/key_request | g:降速 h:打招呼 w/a/s/d:移动 x/空格:站立 Ctrl+C:退出
```

## 3 验证主机话题与机载收包

- 打开 **主机终端 C**，核查 `/pc/key_request` 话题列表与实时消息：

```bash
source /opt/ros/foxy/setup.bash
export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"
ros2 topic list | grep pc
ros2 topic echo /pc/key_request
```

- 点选 **主机终端 B** 并确认切换至 **英文输入法**，依次按一下 `h`（打招呼）和 `x`（平衡站立）。此时 **机载 SSH 窗口** 应立即打印出对应 JSON 包，且机器狗全程静止：

```text
收到 ('10.42.0.1', 46778): {"api_id": 1016, "parameter": "{}"}
收到 ('10.42.0.1', 46778): {"api_id": 1002, "parameter": "{}"}
```

> 注意：主机端 `/pc/key_request` 只有在 `go2_key_ros_pub.py` 运行期间才会在 `ros2 topic list` 中显示；如希望在正式动腿前进一步验证底层收发闭环，可发送非法测试号 `api_id=9999`，底层回执 `code=3203` 即证明机载至底层通信正常且不会触发动作。

# 实机底层转发与安全测试

空跑确认话题发布与 UDP 收包无误后，将机载端切换为单网卡转发底层，进行低速真机测试。

## 1 实机运行前安全检查

> 谨慎：正式切入底层转发前必须完成以下全部安全确认：
> 1. 将 Go2 放置于平坦、干燥且周围留足 **1.5 米以上空地** 的地面，清理周边线缆与杂物。
> 2. 使用实体遥控器让机器狗进入 **平稳站立** 状态，并将遥控器拿在手边随时准备接管。
> 3. 在官方 App 中确认 **`sport_mode`** 为蓝色正常运行状态，且已关闭室内易误触发的避障功能。

## 2 机载端切回单网卡并启动转发

- 保持主机终端 A 与终端 B 运行不变，仅在 **机载 SSH 窗口** 按 `Ctrl + C` 停掉纯收包脚本，然后清除冲突环境变量、加载单网卡脚本并启动 `udp_relay.py`：

```bash
unset CYCLONEDDS_URI
source ~/zjjgo2/run.sh
python3 ~/zjjgo2/udp_relay.py
```

- 启动成功会看见以下终端输出：

```text
UDP->底层已启动: 监听 0.0.0.0:15001 -> /api/sport/request
```

## 3 分步低速测试与按键说明

### 3.1 按键功能对照表

| 按键 | 功能说明 | 对应 API ID | 备注 |
| --- | --- | --- | --- |
| `w` / `s` | 前进 / 后退 | `1008` | 初始线速度 `0.3 m/s` |
| `a` / `d` | 左转 / 右转 | `1008` | 初始角速度 `0.5 rad/s` |
| `q` / `e` / `z` / `c` | 组合转向移动 | `1008` | 线速度与角速度叠加 |
| `Shift` + `W/S/A/D/Q/E/Z/C` | 全向平移模式 | `1008` | 大写触发横向平移 |
| `r` / `t` | 同时增 / 减线速度与角速度 10% | 无（本地调速） | 线速度范围 `0.1~1.0 m/s` |
| `f` / `g` | 单独增 / 减线速度 10% | 无（本地调速） | 真机测试建议先按两下 `g` 降速 |
| `v` / `b` | 单独增 / 减角速度 10% | 无（本地调速） | 角速度范围 `0.1~1.5 rad/s` |
| `h` / `k` | 打招呼 / 伸展 | `1016` / `1017` | 适合首次原地动作验证 |
| `n` / `m` | 坐下 / 从坐姿站起 | `1009` / `1010` | 原地姿态切换 |
| `y` / `u` | 舞蹈 1 / 舞蹈 2 | `1022` / `1023` | 需预留充足空间 |
| `j` | 向前跳 | `1031` | 室内严禁误触 |
| `x` / `空格` / 其他键 | 立即停止并恢复平衡站立 | `1002` | 每次移动后及时点按停稳 |

> "g"：降低线速度；
> "h"：原地打招呼测试；
> "w/s/a/d"：前后移动与左右转向；
> "x" 或 "空格"：立即停止并平衡站立；
> "Ctrl+C"：发送平衡站立指令并安全退出节点；

### 3.2 推荐测试与安全收尾顺序

- 点选 **主机终端 B**（确保处于 **英文输入法**），按以下顺序操作：
  1. 先按两下 **`g`** 将线速度降至约 `0.24 m/s`。
  2. 单击一下 **`h`** 测试打招呼，确认机载窗口打印 `[传底层] api_id=1016 param={}` 且机器狗抬腿挥手正常。
  3. 轻点一下 **`w`** 前进一小步，紧接着按 **`x` 或空格** 站稳；再依次轻点 `s`、`a`、`d` 测试，每次动作后立即按 `x` 或空格停稳。
- 实验结束后严格按以下顺序退出：
  1. 在 **主机终端 B** 先按 **`x` 或空格** 让机器狗站稳，再按 `Ctrl + C` 退出键盘节点（终端打印 `已安全退出。`）。
  2. 在 **主机终端 A** 按 `Ctrl + C` 停止 `pc_ros_to_udp.py`。
  3. 在 **机载 SSH 窗口** 按 `Ctrl + C` 停止 `udp_relay.py`。

# 常见故障排除与动捕扩展

本模块汇总实验过程中出现的典型报错与解决方案，并给出后续接入动捕 ROS 话题的扩展思路。

## 1 常见故障排除

### 1.1 启动热点提示 `enp5s0` 不匹配

- 执行 `nmcli connection up Hotspot` 时出现以下报错：

```text
Error: Connection activation failed: No suitable device found for this connection (device enp5s0 not available because profile is not compatible with device (mismatching interface name)).
```

> 解决方法：
> 1. 执行 `nmcli device status` 确认实际无线网卡名（本机为 `wlo1`）。
> 2. 删除绑定错误的旧配置并重建热点：`nmcli connection delete Hotspot` 后运行 `nmcli device wifi hotspot ifname wlo1 ssid "Go2_Hotspot" password "12345678"`。

### 1.2 主机节点或 `ros2 topic list` 报 `enp5s0` 不可用

- 运行 Python 节点或新开终端执行 `ros2 topic list` 时出现以下报错：

```text
python3: enp5s0: does not match an available interface.
[ERROR] [rmw_cyclonedds_cpp]: rmw_create_node: failed to create domain, error Error
rclpy._rclpy_pybind11.RCLError: Unknown error creating node: rcl node's rmw handle is invalid
```

> 解决方法：
> 1. 原因在于当前终端加载的 `CYCLONEDDS_URI` 仍指向未插线的 `enp5s0`。
> 2. 在当前终端显式执行 `export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"` 后重新运行命令。

### 1.3 `CYCLONEDDS_URI` XML 解析报错

- 手动在命令行输入 XML 字符串时出现如下报错：

```text
python3: config /: parser error expecting '<' () at line 1 (CYCLONEDDS_URI+143 line 1)
```

> 解决方法：
> 1. 常见原因为单引号未闭合、末尾多出 `>` 或标签拼写错误。
> 2. 不要手动敲行内 XML，统一使用 `~/cyclone_wlo1.xml` 文件并通过 `export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"` 引用。

## 2 后续接入动捕 ROS 话题思路

本套桥接骨架可直接复用于光学动捕（如 Nokov / VRPN）闭环控制：

- **情况一（动捕 ROS 节点跑在 Ubuntu 主机端）**：保持动捕节点在主机本地正常发布位姿/速度话题（如 `PoseStamped` / `Odometry` / `Twist`），在主机侧编写控制器节点订阅动捕话题并计算期望运动指令，再复用 `pc_ros_to_udp.py` 通过 UDP `15001` 发往机载端 `udp_relay.py`，继续隔离主机与底层 DDS。
- **情况二（动捕 ROS 节点直接跑在 Go2 机载端）**：无需经过 Wi-Fi 热点绕行，直接在机载单网卡环境下订阅本地动捕话题并发布至 `/api/sport/request` 即可。

> 注意：无论是键盘控制还是后续动捕闭环控制，每次修改控制逻辑上真机前，均建议先按第四章流程做纯 UDP 空跑打印验证帧率、坐标系符号与数值范围，确认无误后再切入机载底层转发。