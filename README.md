# ROS 2 学习与实机实验

WSL2 + Ubuntu 24.04 + ROS 2 Jazzy 环境下的学习记录，包含自己写的节点、一次真实摄像头接入实验，以及完整的排错过程。

> 本仓库只保留**自己写的代码与文档**。课程示例源码（古月居 `ros2_21_tutorials`）不入库，仅在文档中注明出处与修改点。

---

## 环境

| 项目 | 版本 |
|---|---|
| Windows | 11 25H2（build 26200） |
| WSL | 2.6.2.0，内核 6.6.87 |
| Ubuntu | 24.04.5 LTS |
| ROS 2 | Jazzy（320 个包，含 desktop、ros_gz） |
| 编辑器 | VS Code + WSL Remote + clangd |
| 图形 | WSLg（Gazebo 走 D3D12 硬件渲染） |

---

## 目录

```
ros2-learning/
├─ 01-my-first-pkg/          自写的 C++ 功能包（定时发布者节点）
└─ 02-vision-red-detection/  笔记本摄像头接入 + HSV 红色区域识别
```

| 子项目 | 内容 | 状态 |
|---|---|---|
| [01-my-first-pkg](01-my-first-pkg/) | C++ 节点：每 1 秒发布一条字符串；CMake 配置 | ✅ 编译运行通过 |
| [02-vision-red-detection](02-vision-red-detection/) | usbipd 转发 USB 摄像头 → `/dev/video0` → OpenCV 实时 HSV 红色分割 | ✅ 定性验证通过 |

---

## 时间线

| 日期 | 内容 |
|---|---|
| 11-17 | 搭建 WSL + Ubuntu 24.04 + ROS 2 Jazzy（rootfs 导入路线） |
| 11-18 | turtle 仿真、rqt 工具、话题/服务实验 |
| 11-19 | tmux、Linux 权限、git 身份配置、VS Code 接 WSL |
| 11-20 | 第一个 C++ 功能包与发布者节点 |
| 11-22 | 接入真实摄像头，ROS 2 视觉节点，HSV 调参 |

---

## 说明

- 所有实验记录都是**实际操作的产物**，包括失败和返工（见各子目录的「排错记录」）
- 数字和结论只写验证过的部分，未量化的地方明确标注"定性"
- 这是学习仓库，不是产品代码；代码风格和工程化程度会随学习进度改进

---

## 参考

- [古月居 ROS 2 教程](https://github.com/guyuehome/ros2_21_tutorials)
- [微软 WSL USB 接入文档](https://learn.microsoft.com/zh-cn/windows/wsl/connect-usb)
- [OpenCV HSV 颜色空间说明](https://docs.opencv.org/4.13.0/df/d9d/tutorial_py_colorspaces.html)
