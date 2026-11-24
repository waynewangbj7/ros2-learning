# 03-dual-node-vision

把第 8 讲的图像链路跑通，并把**相机发布端**从课程自带的 OpenCV 节点换成 `usb_cam`——用来验证"节点级模块化"这件事。

实验编号：`ROS2-B-261123-001`　状态：两种发布端均完成定性端到端验证

---

## 链路

```
topic_webcam_pub（OpenCV）  或  usb_cam_node_exe（二选一）
        ↓  /image_raw   [sensor_msgs/msg/Image]
topic_webcam_sub
        ↓  CvBridge → bgr8 → HSV 红色筛选
   画面窗口（绿色轮廓）
```

**关键点**：识别端 `topic_webcam_sub` 一个字没改。换掉发布端后整个链路照常工作——因为它只依赖 `/image_raw` 这个**话题接口**，不关心谁在发。

---

## 验证结果

| 阶段 | 证据 |
|---|---|
| WSL 取帧 | 重接后 `/dev/video0` 可用；V4L2 + MJPG + 320×240 连续读到 5 帧，每帧形状 `(240, 320, 3)` |
| 课程 OpenCV 双节点 | `ros2 topic echo` 收到 **320×240、`bgr8`、`step: 960`、像素 230400 字节**；订阅端弹窗圈出红色区域 |
| 换成 `usb_cam` | 启动日志记录 `/dev/video0`、320×240、`mmap`、`mjpeg2rgb`，请求 30 FPS；两端同时运行后实时画面正常 |

⚠️ 截图是 **OpenCV 双节点阶段**的证据，不是换装 `usb_cam` 后的截图。`usb_cam` 阶段只有启动日志和人工确认，**没有记录单帧 echo、没测帧率/延迟/识别率**。

---

## 运行方法（usb_cam 方案）

### 终端 A：相机发布端

```bash
source /opt/ros/jazzy/setup.bash
ros2 run usb_cam usb_cam_node_exe --ros-args \
  -p video_device:=/dev/video0 \
  -p pixel_format:=mjpeg2rgb \
  -p image_width:=320 \
  -p image_height:=240
```

（`sudo apt install ros-jazzy-usb-cam` 只需装一次）

### 终端 B：识别订阅端

```bash
source /opt/ros/jazzy/setup.bash
source ~/guyue_ros2_ws/install/local_setup.bash
ros2 run learning_topic topic_webcam_sub
```

### 验证

```bash
# 第三个终端，确认话题上真的有帧
ros2 topic echo /image_raw sensor_msgs/msg/Image --once --timeout 15 --no-arr
```

窗口里画面跟着动作变化、红色物块被圈出 = 通过。

**顺序很重要**：先启动发布端，等它稳定，再启动订阅端。反过来容易看到"订阅端没输出"，误以为卡住。

---

## 得到的认知

| 认知 | 说明 |
|---|---|
| **话题接口就是解耦点** | 换相机、换发布端，识别端不用改——这是 ROS 最有价值的设计 |
| **`usb_cam` 不是通用驱动** | 换别的相机仍需匹配设备、格式、话题名、消息类型、可转换的编码、兼容的 QoS |
| **`Publisher count: 0` 不代表没在跑** | 只说明查询那一刻没发现发布者；各为 `1` 也不证明有帧 |
| **课程代码的日志可能误导** | 发布端 `Publishing video frame` 写在取帧成功判断**之外**，不能当"有画面"的证据，得用 `topic echo` 验 |
| **33 ms 是定时器间隔，不是帧率** | 不能拿它当实测 FPS |
