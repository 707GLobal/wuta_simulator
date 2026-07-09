# workflow of vehicle_model and can_simulator

## 一、节点概览

```
包: wuta_simulator (Python, ament_python)
├── vehicle_model    ── 自行车运动学模型，接收控制指令，模拟车辆运动
└── can_simulator    ── 模拟 CAN 总线转发，将仿真速度转发给下层定位
```

---

## 二、vehicle_model 节点

### 订阅

| Topic | 类型 | 说明 |
|-------|------|------|
| `/control/command` | `autoware_msgs/Command` | 控制指令 (speed, angle, dv_state) |

### 发布

| Topic | 类型 | 频率 | 说明 |
|-------|------|------|------|
| `/sim/ground_truth` | `nav_msgs/Odometry` | 50Hz | 车辆 ground truth 位姿 |
| `/sim/velocity` | `geometry_msgs/TwistStamped` | 50Hz | 车辆 ground truth 速度 |

### Odometry 消息结构 (`/sim/ground_truth`)

```
header.frame_id = "map"
header.stamp    = now

pose.pose.position.x = self.x
pose.pose.position.y = self.y
pose.pose.orientation.z = sin(yaw / 2)
pose.pose.orientation.w = cos(yaw / 2)
```

### TwistStamped 消息结构 (`/sim/velocity`)

```
header.frame_id = "base_link"

twist.linear.x  = self.speed           (前进速度, m/s)
twist.angular.z = v·tan(δ) / L         (横摆角速度, rad/s)
```

### 运动学模型

自行车模型更新公式（50Hz 离散化）：

```
x    += v · cos(yaw) · dt
y    += v · sin(yaw) · dt
yaw  += v · tan(δ) / L · dt
```

其中：
- `v` — 纵向速度 (m/s)，来自控制指令 `speed`
- `δ` — 转向角 (rad)，来自控制指令 `angle`，经 ±25° 限幅
- `L` — 轴距 `wheel_base`，默认 1.53m
- `dt` — 时间步长，默认 0.02s

---

## 三、can_simulator 节点

### 订阅

| Topic | 类型 | 说明 |
|-------|------|------|
| `/sim/velocity` | `geometry_msgs/TwistStamped` | 接收车辆 ground truth 速度 |

### 发布

| Topic | 类型 | 说明 |
|-------|------|------|
| `/localization/velocity` | `geometry_msgs/TwistStamped` | 模拟 CAN 总线输出的车速信号 |

### 行为

纯透传转发，重新打时间戳：
```
/sim/velocity ──→ [CAN Bus Simulator] ──→ /localization/velocity
```

所有 `twist.linear.*` 和 `twist.angular.*` 字段原样传递。

---

## 四、数据流图

```
┌──────────────────────────────────────────────────────┐
│                  wuta_simulator 包                    │
│                                                      │
│  planner / controller                                │
│       │                                              │
│       │  /control/command (autoware_msgs/Command)    │
│       ▼                                              │
│  ┌────────────────┐                                  │
│  │ vehicle_model  │─── /sim/ground_truth (Odometry)  │
│  │ (运动学模拟)   │─── /sim/velocity (TwistStamped)  │
│  └───────┬────────┘                                  │
│          │                                           │
│          ▼                                           │
│  ┌────────────────┐                                  │
│  │ can_simulator  │─── /localization/velocity        │
│  │ (CAN 转发)     │    (TwistStamped)                │
│  └────────────────┘                                  │
│          │                                           │
│          ▼                                           │
│    定位 / 控制节点                                    │
└──────────────────────────────────────────────────────┘
```

---

## 五、节点参数

### vehicle_model 参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `wheel_base` | 1.53 m | 车辆轴距 |
| `max_steer_angle` | 25.0 deg | 最大转向角 |
| `dt` | 0.02 s (50Hz) | 仿真步长 |
| `start_x` | 0.0 | 初始 x 位置 |
| `start_y` | 0.0 | 初始 y 位置 |
| `start_yaw` | 0.0 | 初始偏航角 |

配置文件：`config/vehicle_model.yaml`

```yaml
/vehicle_model:
  ros__parameters:
    wheel_base: 1.53
    max_steer_angle: 25.0
    dt: 0.02
    start_x: 0.0
    start_y: 0.0
    start_yaw: 0.0
```

### can_simulator 参数

无（纯透传，无需配置）

---

## 六、消息类型依赖

| 消息 | 来源包 | 依赖关系 |
|------|--------|----------|
| `autoware_msgs/Command` | `autoware_msgs` | `vehicle_model` 订阅 |
| `nav_msgs/Odometry` | `nav_msgs` | `vehicle_model` 发布 |
| `geometry_msgs/TwistStamped` | `geometry_msgs` | `vehicle_model` 发布，`can_simulator` 收发 |

---

## 七、快速验证

### 启动节点

```bash
# 终端 1
python3 vehicle_model.py

# 终端 2
python3 can_simulator.py
```

### 发送控制指令

```bash
ros2 topic pub /control/command autoware_msgs/msg/Command "{speed: 1.0, angle: 5.0}" --rate 10
```

### 监听输出

```bash
# 查看 ground truth 位姿
ros2 topic echo /sim/ground_truth

# 查看仿真速度
ros2 topic echo /sim/velocity

# 查看 CAN 转发后的速度
ros2 topic echo /localization/velocity
```

### 直行验证

```bash
# 发直线指令（角度=0）
ros2 topic pub /control/command autoware_msgs/msg/Command "{speed: 2.0, angle: 0.0}" -1
# 期望: x 递增, y≈0, yaw≈0
```

### 转弯验证

```bash
# 发转弯指令
ros2 topic pub /control/command autoware_msgs/msg/Command "{speed: 1.0, angle: 10.0}" -1
# 期望: x/y/yaw 都变化，轨迹为圆弧
```
