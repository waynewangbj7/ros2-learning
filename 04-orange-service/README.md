# 04-orange-service

第 9 讲：把图像话题接到**服务端**，用 ROS 2 服务查询目标在画面中的像素坐标。

实验编号：`ROS2-B-261123-002`　状态：实时橙色检测、单次坐标查询、画面实时叠字均已跑通

---

## 三个终端

```
终端 A：usb_cam ── /image_raw ──▶ 终端 B：service_object_server（每帧检测 + 叠字）
终端 C：service_object_client ── /get_target_position (get: true) ──▶ 终端 B
终端 C：打印 x, y 后退出 ◀── 返回最近一帧的检测坐标
```

| 角色 | 节点 | 作用 |
|---|---|---|
| 发布端 | `usb_cam_node_exe` | 持续发布 `/image_raw` |
| 服务端 | `service_object_server` | 每帧 HSV 检测、画轮廓、叠字，维护"最近一次检测坐标" |
| 客户端 | `service_object_client` | 运行一次、查询一次、打印后退出 |

服务接口：`learning_interface/srv/GetObjectPosition`

---

## 结果

实测客户端返回：**`x: 214, y: 157`**

| 说明 | 内容 |
|---|---|
| 坐标含义 | **320×240 图像中的像素位置**，不是距离，也不是三维位置 |
| 截图 | [橘子轮廓](orange-fruit.png)、[橙色物块](orange-object.png)（叠字前的检测效果） |
| 叠字 | 以人工确认为准，截图里没有 |

---

## 参数

| 项目 | 值 |
|---|---|
| HSV 初版 | `[10, 120, 100]` – `[25, 255, 255]` |
| 后续调整 | 只把亮度下限 `100 → 80`，保留色相与饱和度 |
| 无目标约定 | 返回 `(0, 0)` |
| 多轮廓处理 | 保存循环中**最后一个**，不保证选中最大或最近的目标 |

> 这次调参**没有定量对照**，不能写成"识别率提升"。

---

## 代码位置（容易错的地方）

叠字的位置要求很严格：

```
object_detect()
├─ for 循环（遍历轮廓）
│   └─ if 有轮廓:
│        └─ putText(...)        ← 比 if 再深一级
├─ imshow(...)                  ← 与 if 同级
└─ waitKey(...)                 ← 与 if 同级
```

放错层级的典型报错：

| 报错 | 原因 |
|---|---|
| `IndentationError` | `putText` 没比 `if` 多缩进一级 |
| `NameError: name 'self' is not defined` | `if` 跑到了方法外 |
| `colcon` 显示 `Finished` 但运行报错 | 编译通过 ≠ 语法正确，要单独看 stderr |

**改代码只改 `src/`**，`install/` 里是编译副本，改了会被下次编译覆盖。

---

## 运行

### 终端 A：相机

```bash
source /opt/ros/jazzy/setup.bash
ros2 run usb_cam usb_cam_node_exe --ros-args \
  -p video_device:=/dev/video0 -p pixel_format:=mjpeg2rgb \
  -p image_width:=320 -p image_height:=240
```

### 终端 B：服务端

```bash
cd ~/guyue_ros2_ws
source /opt/ros/jazzy/setup.bash
source install/local_setup.bash

# 改过 src 才需要：先语法检查再编译
python3 -m py_compile src/ros2_21_tutorials/learning_service/learning_service/service_object_server.py
colcon build --packages-select learning_service

ros2 run learning_service service_object_server
```

### 终端 C：查一次坐标

```bash
source /opt/ros/jazzy/setup.bash
source ~/guyue_ros2_ws/install/local_setup.bash
ros2 run learning_service service_object_client
```

---

## 边界

| 做到了 | 没做 |
|---|---|
| 橙色区域实时筛选与轮廓 | 区分橘子与其他橙色物体 |
| 返回单点像素坐标 | 距离/三维位置估计 |
| 服务请求-应答跑通 | 连续跟踪、多目标、跟踪 ID |
| 画面实时叠字 | 准确率/召回率、帧率测量 |

**相机停帧后服务仍可能返回上一次保存的坐标**——这是"最近一帧"设计的副作用，用的时候要留意。
