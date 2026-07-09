## 架构设计

新建 1 个包，所有脚本用 Python 3 实现（快速开发，不编译）。

```Plain Text
src/simulation/wuta_simulator/
├── package.xml
├── setup.py
├── config/
│   └── simulator.yaml
├── launch/
│   └── simulator.launch.py     # 一键启动仿真器 + FSD 管线
├── wuta_simulator/
│   ├── __init__.py
|   |—— track_loader.py
│   ├── vehicle_model.py        # 自行车运动学模型
│   ├── lidar_simulator.py      # 射线投射模拟 LiDAR
│   ├── ins_simulator.py        # 模拟 CG-410 INS
│   └── can_simulator.py        # 模拟 CAN 车速上报
└── tracks/
    ├── trackdrive.yaml         # 赛道追逐赛道定义
    ├── skidpad.yaml            # 八字绕桩赛道定义
    └── acceleration.yaml       # 直线加速赛道定义
```
