# carla_APIcode
# Easy CARLA Tutorial
一个面向初学者的CARLA仿真器入门教程，聚焦传感器数据采集（摄像头/IMU/GNSS/LiDAR）、车辆控制与坐标变换核心能力，适配Jupyter Notebook和Python脚本双运行环境，基于[Yang2581/easy_carla_tutorial](https://github.com/Yang2581/easy_carla_tutorial)优化适配。

## 📋 目录
- [环境要求](#环境要求)
- [兼容性说明](#兼容性说明)
- [核心功能](#核心功能)
- [快速开始](#快速开始)
- [关键模块详解](#关键模块详解)
- [资源清理规范](#资源清理规范)
- [常见问题排查](#常见问题排查)
- [致谢](#致谢)

## 🛠️ 环境要求
### 基础依赖
| 组件               | 版本要求                  | 补充说明                     |
|--------------------|---------------------------|------------------------------|
| CARLA 仿真器       | 0.9.15 / 0.9.16（稳定版） | 0.10.0为测试版，存在兼容性问题 |
| Python             | 3.10 / 3.11 / 3.12        | 0.9.16版本推荐Python 3.12    |
| 显卡显存           | ≥ 6GB                     | 低画质下Arc 140t/GTX 1050Ti可运行 |
| 操作系统           | Ubuntu 20.04/22.04（推荐）<br>Windows 10/11 | Windows需注意CV2窗口阻塞问题 |

### Python包依赖
执行以下命令安装依赖包：
```bash
pip install carla numpy opencv-python open3d matplotlib
```

## ⚠️ 兼容性说明
### 已知不兼容场景
1. **CARLA 0.10.0+版本**：底层API重构导致核心功能失效
   - Jupyter Notebook运行时内核直接崩溃
   - 客户端连接、车辆生成、传感器回调等功能均无法稳定使用
   - 与现有开源教程/资源生态脱节，不建议使用
2. **Python 3.9及以下**：易出现CARLA API导入错误、类型不匹配问题
3. **Windows系统**：CV2窗口显示摄像头流时可能出现阻塞，建议替换为matplotlib可视化

### 推荐最优环境组合
```
CARLA 0.9.16 + Python 3.12 + Ubuntu 22.04
```

## ✨ 核心功能
1. **CARLA基础交互**
   - 客户端-服务器连接与版本验证
   - 地图加载/切换（默认Town04）
   - 观察者视角坐标实时打印
   - 车辆生成（特斯拉Model3）与自动驾驶模式启用
2. **多传感器数据采集**
   - 四路摄像头（前/后/左/右）：实时图像拼接与可视化
   - IMU传感器：加速度/陀螺仪/指南针数据采集（带噪声模拟）
   - GNSS传感器：经纬度/海拔数据采集（带定位误差模拟）
   - 64线LiDAR：点云数据采集、可视化与PCD格式保存
3. **车辆运动学计算**
   - 全局速度（m/s → km/h）转换与打印
   - 全局坐标系→车辆本地坐标系速度分解（纵向/横向/垂直）

## 🚀 快速开始
### 1. 启动CARLA仿真器
#### Ubuntu系统
```bash
cd /path/to/CARLA_0.9.16
# 低画质模式适配低配显卡（如Arc 140t/GTX 1050Ti）
./CarlaUE4.sh -quality-level=Low
```

#### Windows系统
```bash
cd C:\path\to\CARLA_0.9.16
CarlaUE4.exe
```

### 2. 运行教程代码
#### 方式1：Jupyter Notebook（推荐调试）
```bash
jupyter notebook easy_carla_tutorial.ipynb
```
按单元格顺序运行，可逐段验证每一步功能（如先连接客户端，再生成车辆，最后创建传感器）。

#### 方式2：Python脚本
```bash
python easy_carla_tutorial.py
```
> 注意：运行前需手动注释/取消注释无限循环逻辑（如观察者坐标打印），避免程序卡死。

## 📌 关键模块详解
### 1. 车辆生成与自动驾驶
```python
# 筛选特斯拉Model3蓝图
vehicle_blueprints = world.get_blueprint_library().filter('vehicle.tesla.model3')
# 定义生成位姿（Town04地图有效坐标）
transform = carla.Transform(
    carla.Location(x=214, y=-173, z=0.5),
    carla.Rotation(pitch=3.73, yaw=176.68, roll=7.96)
)
# 生成车辆并启用自动驾驶
ego_vehicle = world.spawn_actor(vehicle_blueprints[0], transform)
ego_vehicle.set_autopilot(True)
```

### 2. 四路摄像头配置
- 支持自定义分辨率（默认400×300）、FOV（110°）、采样率（20Hz）
- 摄像头位姿可灵活调整（基于车辆坐标系：X前/Y右/Z上）
- 实时拼接四路图像并通过CV2窗口显示，按`q`键退出

### 3. IMU/GNSS传感器
- IMU：50Hz采样率，支持加速度/陀螺仪噪声配置
- GNSS：10Hz采样率，支持经纬度/海拔噪声模拟
- 回调函数自动解析传感器数据，存入字典便于后续调用

### 4. 车辆速度坐标变换
核心函数实现全局速度到车辆本地速度的转换：
```python
def get_vehicle_speeds(vehicle):
    vel_world = vehicle.get_velocity()
    vel_vector = np.array([vel_world.x, vel_world.y, vel_world.z])
    yaw = math.radians(vehicle.get_transform().rotation.yaw)
    
    # 旋转矩阵：世界坐标系 → 车辆坐标系
    R = np.array([
        [math.cos(yaw), math.sin(yaw), 0],
        [-math.sin(yaw), math.cos(yaw), 0],
        [0, 0, 1]
    ])
    
    vel_local = R @ vel_vector
    return vel_local[0], vel_local[1], vel_local[2]  # 纵向/横向/垂直速度
```

### 5. LiDAR点云处理
封装`LidarManager`类实现：
- 64线LiDAR配置（360°水平FOV、100米探测范围）
- 点云数据实时解析（N×4格式：x/y/z/intensity）
- 点云可视化与PCD格式保存（支持强度值着色）

## 🧹 资源清理规范
CARLA仿真器中Actor（车辆/传感器）未正确销毁会导致内存泄漏、模拟器崩溃，需严格执行以下清理逻辑：
```python
# 摄像头清理
for cam in cameras.values():
    cam.stop()
    cam.destroy()

# IMU/GNSS清理
for sensor in sensors:
    sensor.stop()
    sensor.destroy()

# LiDAR清理
lidar_mgr.destroy()

# 车辆清理
ego_vehicle.destroy()

# CV2窗口清理
cv2.destroyAllWindows()
```

## 🐞 常见问题排查
### 1. 客户端连接失败
- 检查CARLA模拟器是否已启动
- 验证端口2000是否被占用（Ubuntu：`netstat -tulpn | grep 2000`）
- 增加连接超时时间：`client.set_timeout(20.0)`

### 2. CV2窗口冻结（Jupyter环境）
替换CV2可视化为matplotlib：
```python
import matplotlib.pyplot as plt
plt.imshow(cv2.cvtColor(combined_image, cv2.COLOR_BGR2RGB))
plt.axis('off')
plt.show()
```

### 3. 车辆生成失败
使用地图预定义有效生成点：
```python
spawn_points = world.get_map().get_spawn_points()
transform = spawn_points[0]  # 选用第一个有效生成点
```

### 4. LiDAR点云为空
- 确认Open3D已正确安装：`pip install open3d`
- 检查LiDAR参数配置（如range/channel不能设为0）

## 🙏 致谢
- [CARLA官方文档](https://carla.readthedocs.io/)
- [Yang2581/easy_carla_tutorial](https://github.com/Yang2581/easy_carla_tutorial)
