# GS-LIVO 实战复现指南
本节将指导你如何在没有真实硬件设备的情况下，使用官方开源数据集，在自己的个人电脑上从零跑通 GS-LIVO（3DGS + 激光惯性视觉里程计）。
## 1. 硬件与系统要求 (Prerequisites)
由于 3DGS 渲染极其消耗显存与算力，本书强烈建议使用以下配置：
- 操作系统：Ubuntu 20.04 (或 Ubuntu 18.04 / 22.04 配合对应的 ROS 版本)
- ROS 版本：ROS Noetic (对应 Ubuntu 20.04)
- 显卡要求：NVIDIA GPU，显存 $\ge$ 8GB（推荐 RTX 3060 及以上）
- CUDA 版本：CUDA 11.3 及以上版本
## 2. 环境配置与编译 (Installation)
GS-LIVO 是一个典型的“C++ (ROS前端) + Python (3DGS后端)”混合工程。我们需要分别配置这两部分。
### 2.1 编译 ROS 工作空间 (前端 LIO)
打开终端，安装必要的依赖并编译工作空间：
```bash
# 1. 安装基础依赖
sudo apt-get install ros-noetic-cv-bridge ros-noetic-pcl-conversions
sudo apt-get install libpcl-dev libopencv-dev

# 2. 创建工作空间并克隆代码
mkdir -p ~/gs_livo_ws/src
cd ~/gs_livo_ws/src
git clone https://github.com/HKUST-Aerial-Robotics/GS-LIVO.git

# 3. 编译 C++ 源码
cd ~/gs_livo_ws
catkin_make
# 将环境变量写入 bashrc，方便后续调用
echo "source ~/gs_livo_ws/devel/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 2.2 配置 Python 虚拟环境 (后端 3DGS)
为了不污染系统环境，我们使用 Conda 来管理 3DGS 所需的 PyTorch 与可微光栅化算子库：
```bash
# 1. 创建名为 gs_livo 的虚拟环境
conda create -n gs_livo python=3.8 -y
conda activate gs_livo

# 2. 安装 PyTorch (请根据你的 CUDA 版本调整 index-url)
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu113

# 3. 进入项目目录，安装 Python 依赖库
cd ~/gs_livo_ws/src/GS-LIVO
pip install -r requirements.txt
```
## 3. 下载测试数据集 (Download Datasets)
你不需要购买任何传感器！GS-LIVO 作者团队（港科大）已经为大家录制好了高质量的数据包。请前往作者的官方 GitHub 仓库（或他们提供的 Google Drive / 百度网盘链接），下载示例数据集（例如 ```campus_seq.bag``` 或其他场景）。

_建议将下载好的 ```.bag ```文件统一存放在``` ~/gs_livo_ws/data/ ```目录下，方便管理。_

## 4. 运行系统 (Run the System)
跑通系统通常需要打开三个终端，模拟“启动建图算法”与“开启传感器输入”的过程。

### 终端 1：启动 GS-LIVO 核心节点与 Rviz

```bash
cd ~/gs_livo_ws
source devel/setup.bash
conda activate gs_livo
# 启动 launch 文件（注意：请根据你下载的数据集替换 launch 文件名）
roslaunch gs_livo run_campus.launch
```

_此时，电脑屏幕上会弹出一个 Rviz 窗口（用于显示点云和相机轨迹）以及一个 3DGS 的实时渲染窗口（起初是黑屏的）。_

### 终端 2：播放数据集 (模拟硬件传感器发数据)
打开一个新的终端，播放刚才下载的 Rosbag：

```bash
cd ~/gs_livo_ws/data/
# 播放数据包，建议加上 --clock 参数同步时间
rosbag play campus_seq.bag --clock
```

_奇迹发生了！随着 bag 包的播放，你会看到 Rviz 中开始绘制出高精度的激光里程计轨迹，而 3DGS 窗口中则像“魔法”一样，伴随着相机的移动，从稀疏的点云中逐渐生长出极其逼真、带有照片级纹理的 3D 高斯场景。_
## 5. 常见报错与排错指南 (Troubleshooting)

为了防止大家“从入门到放弃”，整理了以下常见坑点：
#### 1. 报错提示 CUDA / PyTorch 版本不匹配
- 原因：安装的```submodules/diff-gaussian-rasterization``` 是需要 C++ 和 CUDA 联合编译的。如果系统 ```nvcc```版本和 PyTorch 版本不一致就会报错。
- 解决：检查 ```nvcc -V```，确保与 conda 中安装的 PyTorch CUDA 版本完全对应。
#### 2. 跑了几十秒后，程序突然崩溃 / Out of Memory (OOM)
- 原因：3DGS 在建图过程中，高斯球的数量会不断分裂（Split）和克隆（Clone），导致显存爆炸。
- 解决：在配置文件（```.yaml```）中，尝试调大 ```prune_threshold```，或者调小图像的分辨率，以限制高斯球的最大数量。
#### 3. 画面一片模糊或全黑，轨迹乱飞
- 原因：电脑太卡导致 ROS 掉帧，或者没加``` --clock``` 导致时间戳错乱。
- 解决：尝试降低 rosbag 播放速度，例如``` rosbag play campus_seq.bag --clock -r 0.5```（以 0.5 倍速播放）。











