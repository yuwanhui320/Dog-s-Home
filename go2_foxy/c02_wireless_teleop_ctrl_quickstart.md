# Unitree Go2 无线键盘控制与 ROS 2 桥接精简手册

- **通信链路**：主机键盘节点（`/pc/key_request`） → 主机 ROS 转 UDP（`15001`） → Go2 机载单网卡转发（`/api/sport/request`）
- **环境前提**：Ubuntu 20.04（ROS 2 Foxy），所有主机终端均需退出 Conda 环境（`conda deactivate`）；调试期间 Go2 仅保持 **1 个 SSH 窗口**。

---

## 1. 网络连接与主机 DDS 网卡绑定

### 1.1 主机开启热点并确认 Go2 连通
在 Ubuntu 主机终端用无线网卡 `wlo1` 开启热点，待 Go2 连上（约 30 秒）后查询其 IP 并测试连通性：

```bash
nmcli connection delete Hotspot 2>/dev/null || true
nmcli device wifi hotspot ifname wlo1 ssid "Go2_Hotspot" password "12345678"

# 确认主机网关为 10.42.0.1，并查看 Go2 分配到的 IP（默认以 10.42.0.211 为例）
ip a show wlo1
ip neigh | grep 10.42.0
ping -c 3 10.42.0.211
```

### 1.2 生成主机 CycloneDDS 配置文件（仅首次执行）
显式绑定 `wlo1`，避免无有线连接时因误绑 `enp5s0` 报错：

```bash
cat << 'EOF' > ~/cyclone_wlo1.xml
<?xml version="1.0" encoding="UTF-8" ?>
<CycloneDDS xmlns="https://cdds.io/config">
  <Domain id="any">
    <General>
      <Interfaces>
        <NetworkInterface name="wlo1" priority="default" multicast="default" />
      </Interfaces>
    </General>
  </Domain>
</CycloneDDS>
EOF
```

---

## 2. 部署控制脚本（仅首次执行）

### 2.1 主机端：键盘 ROS 2 发布节点 `~/go2_key_ros_pub.py`

```bash
cat << 'EOF' > ~/go2_key_ros_pub.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, sys, termios, threading, tty
import rclpy
from rclpy.node import Node
from unitree_api.msg import Request

MOVE, BALANCE_STAND = 1008, 1002
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
    "h": 1016, "j": 1031, "k": 1017,
    "n": 1009, "m": 1010, "y": 1022, "u": 1023,
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

    lin_speed, ang_speed = 0.3, 0.5
    print("[键盘节点已启动] 话题: /pc/key_request | g:降速 h:打招呼 w/a/s/d:移动 x/空格:站立 Ctrl+C:退出")

    try:
        while rclpy.ok():
            key = get_key()
            if key == "\x03":
                break
            if key in SPEED_KEYS:
                lin_speed = max(0.1, min(1.0, lin_speed * SPEED_KEYS[key][0]))
                ang_speed = max(0.1, min(1.5, ang_speed * SPEED_KEYS[key][1]))
                print(f"\r[速度] 线={lin_speed:.2f} m/s, 角={ang_speed:.2f} rad/s          ", end="", flush=True)
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
                api_id, param = ACTION_KEYS[key], "{}"
            else:
                api_id, param = BALANCE_STAND, "{}"

            msg = Request()
            msg.header.identity.api_id = api_id
            msg.parameter = param
            pub.publish(msg)
            print(f"\r[发布] api_id={api_id} param={param}          ", end="", flush=True)
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

### 2.2 主机端：ROS 转 UDP 桥接脚本 `~/pc_ros_to_udp.py`

```bash
cat << 'EOF' > ~/pc_ros_to_udp.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, socket
import rclpy
from rclpy.node import Node
from unitree_api.msg import Request

GO2_IP = "10.42.0.211"    # Go2 IP 变化时同步修改此处
GO2_PORT = 15001

class RosToUdpBridge(Node):
    def __init__(self):
        super().__init__("pc_ros_to_udp_bridge")
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.create_subscription(Request, "/pc/key_request", self.cb, 10)
        self.get_logger().info(f"订阅 /pc/key_request -> UDP {GO2_IP}:{GO2_PORT}")

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

### 2.3 机载端：UDP 转底层转发脚本 `~/zjjgo2/udp_relay.py`
通过 SSH 登录 Go2（出现 ROS 版本选择时输入 `1` 选 Foxy）并创建转发脚本：

```bash
ssh unitree@10.42.0.211
mkdir -p ~/zjjgo2

cat << 'EOF' > ~/zjjgo2/udp_relay.py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json, socket
import rclpy
from rclpy.node import Node
from unitree_api.msg import Request

def main():
    rclpy.init()
    node = Node("go2_udp_relay")
    pub = node.create_publisher(Request, "/api/sport/request", 10)

    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind(("0.0.0.0", 15001))
    sock.settimeout(0.5)
    print("UDP->底层已启动: 监听 0.0.0.0:15001 -> /api/sport/request")

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

---

## 3. 空跑验包（机器狗绝对不动）

每次修改代码上真机前，先执行纯收包测试验证链路：

1. **Go2 SSH 窗口**（只收包打印，不连底层）：
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

2. **主机终端 A**（启动 ROS → UDP 桥接）：
   ```bash
   source /opt/ros/foxy/setup.bash
   source ~/unitree_ros2/setup.sh 2>/dev/null || true
   export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"
   python3 ~/pc_ros_to_udp.py
   ```

3. **主机终端 B**（启动键盘 ROS 2 节点）：
   ```bash
   source /opt/ros/foxy/setup.bash
   source ~/unitree_ros2/setup.sh 2>/dev/null || true
   export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"
   python3 ~/go2_key_ros_pub.py
   ```

4. **核验**：切到 **主机终端 B（英文输入法）**，按一下 `h` 和 `x`。若 Go2 SSH 窗口立刻打印出 `api_id: 1016` 与 `1002`，即证明链路打通（如需查看主机话题，可另开主机终端加载环境后运行 `ros2 topic list | grep pc`）。

---

## 4. 真机运行与安全退出

### 4.1 实机前安全确认
- 放置于平坦空地（周围留出 **1.5 米以上** 无障碍空间）。
- 用遥控器让 Go2 **平稳站立**，遥控器拿在手边随时接管。
- 手机 App 确认 `sport_mode` 为蓝色运行状态，且已关闭避障。

### 4.2 切入底层转发
保持主机终端 A、B 不变，在 **Go2 SSH 窗口**按 `Ctrl + C` 停掉空跑监听，启动单网卡转发：

```bash
unset CYCLONEDDS_URI
source ~/zjjgo2/run.sh
python3 ~/zjjgo2/udp_relay.py
```
这个source的意思是把有线网卡从默认的eth0绑定到eth10，并仅在当前终端生效。
### 4.3 按键速查与操作顺序

| 按键 | 功能 | 说明 |
| --- | --- | --- |
| `w/s/a/d`、`q/e/z/c` | 前后 / 左右转 / 组合移动 | 按住 `Shift` 切换横向平移 |
| `g` / `f`、`b` / `v`、`t` / `r` | 减 / 增线速度、角速度、整体速度 | 每次调整 10% |
| `h` / `k` / `n` / `m` / `y` / `u` | 打招呼 / 伸展 / 坐下 / 站起 / 舞蹈 | `j` 为向前跳，**室内勿按** |
| `x` / `空格` | 立即停止并平衡站立（`1002`） | 每次移动后及时点按停稳 |

- **测试顺序**（主机终端 B，英文输入法）：先按两下 `g` 降速 → 单击 `h` 测打招呼 → 轻点 `w` 立刻按 `x`/空格停稳 → 再测 `s/a/d`。
- **退出顺序**：按 `x`/空格站稳 → 主机终端 B `Ctrl + C` → 主机终端 A `Ctrl + C` → Go2 SSH `Ctrl + C`。

---

## 5. 常见报错速查与动捕扩展

- **热点启动报错 `device enp5s0 not available`**：按第 1.1 节先 `nmcli connection delete Hotspot` 再用 `ifname wlo1` 重建。
- **主机运行节点报 `enp5s0: does not match an available interface`**：当前终端未覆盖 `CYCLONEDDS_URI`，执行 `export CYCLONEDDS_URI="file://$HOME/cyclone_wlo1.xml"` 后重试。
- **收不到包**：用 `ip neigh | grep 10.42.0` 确认 Go2 重启后 IP 是否变化，同步更新 `~/pc_ros_to_udp.py` 中的 `GO2_IP`。
- **后续接入动捕 ROS 话题**：若动捕跑在主机端，直接将控制节点输出发布到 `/pc/key_request`（或复用 `pc_ros_to_udp.py` 发往 `15001`）；若跑在机载端，则在单网卡环境下直接订阅并发布至 `/api/sport/request`。