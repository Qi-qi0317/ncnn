# Build Python Wheel for LoongArch

## 简介

### Loongarch架构概述
LoongArch是完全自主知识产权的指令集架构，其采用基础部分加扩展部分的模块化组织形式，允许CPU根据需求选择实现各扩展部分。LoongArch能够通过二进制翻译的方式兼容MIPS，RISC-V，ARM，x86等指令集的Linux程序。

### Python Wheel包简介
Python wheel包使Python包的端到端的安装相较于下载源码分发包更快，这主要是由于wheel包比源码分发包更小，可以更快地通过网络传输，而且直接从wheel包安装避免了从源码分发包构建包再进行安装的中间过程，从而使得安装变得更快。

### ncnn库简介
ncnn 是一个为手机端极致优化的高性能神经网络前向计算框架。 ncnn 从设计之初深刻考虑手机端的部署和使用。 无第三方依赖，跨平台，手机端 cpu 的速度快于目前所有已知的开源框架。 基于 ncnn，开发者能够将深度学习算法轻松移植到手机端高效执行， 开发出人工智能 APP，将 AI 带到你的指尖。 ncnn 目前已在腾讯多款应用中使用，如：QQ，Qzone，微信，天天 P 图等。

## 环境准备

### 操作系统要求
任何支持QEMU的Linux 发行版。

### 编译工具
#### QEMU模拟器用于Python wheel包的构建以及测试。这里我给出一个简单的操作可以直接快速启动QEMU。
1） 下载QEMU
bash
wget https://download.qemu.org/qemu-9.1.0.tar.xz.sig
tar -xvf qemu-9.1.0.tar.xz
cd qemu-9.1.0
./configure
make
make install
这样你就下载好了一个QEMU软件，但还需要一个操作系统镜像文件。

2） LoongArch 架构的 Linux 发行版镜像文件下载
[http://pkg.loongnix.cn](http://pkg.loongnix.cn)
在这里你可以选择相应的系统镜像文件比如
[https://pkg.loongnix.cn/loongnix/isos/Loongnix-20.6/Loongnix-20.6.netinst.kde.loongarch64.iso](https://pkg.loongnix.cn/loongnix/isos/Loongnix-20.6/Loongnix-20.6.netinst.kde.loongarch64.iso)
现在你可以通过QEMU软件和下载好的iso文件启动一个针对Loongarch架构的QEMU模拟机了。

#### 针对LoongArch架构的交叉编译工具链，该工具链包括编译器、链接器等。
可以直接在LoongArch官网下载相关交叉编译工具链。
注意在下载好之后，需要将交叉编译工具链路径设置成环境变量，保证在后续编译过程中使用交叉编译工具链进行编译工作。
确保CC, CXX, 和 LD 环境变量指向LoongArch交叉编译工具链。
bash
export CC=/path/to/loongarch-gcc
export CXX=/path/to/loongarch-g++
export LD=gcc
export PATH=/path/to/loongarch/python-install/bin:$PATH
（临时覆盖默认的gcc，使其指向loongarch架构的gcc）由于我在设置过程中设置环境变量一直不成功，所以采取这种方式。
bash
sudo ln -sf ~/loongarch-toolchain/bin/loongarch64-linux-gnu-gcc /usr/local/bin/gcc

### Loongarch架构下的Python解释器编译
由于我们需要为loongarch架构构建其对应的python wheel，所以需要使用loongarch架构下的Python解释器来进行wheel包的编译，所以我们需要在我们当前架构下使用交叉编译工具链编译一个loongarch架构的Python解释器来供我们在后续QEMU模拟器中使用。
为了避免不同版本的Python和库的相互影响，我们需要建立一个隔离的Python运行空间来使全局环境不受影响。
bash
python3 -m venv /path/to/your/venv
source /path/to/your/venv/bin/activate

#### 获取Python源代码
bash
git clone https://github.com/python/cpython.git
./configure --build=x86_64-pc-linux-gnu --host=loongarch64-unknown-linux-gnu --prefix=/Path/to/Python CC=loongarch64-linux-gnu-gcc READELF="loongarch64-linux-gnu-readelf" --disable-ipv6 --enable-optimizations
确保使用交叉编译工具链进行编译，即环境变量设置正确，如果设置不成功，可以进行一下类似操作进行明确指出。
make
make install
现在，你已经拥有一个可以在loongarch架构使用的Python解释器啦！你可以在/Python/bin/中找到相应的可执行文件。如果在非Loongarch架构机器上运行，会提示二进制文件不可执行。

### 编译Vulkan SDK
bash
git clone https://github.com/KhronosGroup/Vulkan-Hpp.git
cd Vulkan-Hpp
./vulkansdk build
make install

### ncnn编译
bash
git clone https://github.com/Tencent/ncnn.git
cd ncnn
mkdir build
cmake -DCMAKE_BUILD_TYPE=Release -DNCNN_VULKAN=ON ..
make

### 启动QEMU虚拟机
bash
cd /path/to/qemu
./qemu-system-loongarch64 -cdrom path/to/your.iso
-drive file=/path/to/your/python-interpreter,format=raw,id=python-drive
-drive file=/path/to/your/vulkan-sdk,format=raw,id=vulkan-sdk-drive
-drive file=/path/to/your/ncnn-project,format=raw,id=ncnn-drive
使用 QEMU 的 -drive 参数将包含 Python 解释器、Vulkan SDK和ncnn的文件系统挂载到虚拟机中
#### 配置其他参数（可选）
-m 或 -memory：设置虚拟机的内存大小。例如，-m 2048 将分配 2048MB 的内存。
-smp：设置虚拟机的 CPU 核心数。例如，-smp 4 表示分配 4 个 CPU 核心。
-cpu：指定虚拟机使用的 CPU 类型，例如 nehalem 或 Broadwell。
-cdrom：指定虚拟机使用的 CD-ROM 镜像文件。例如，-cdrom ubuntu.iso。
-boot：设置虚拟机的启动顺序。例如，-boot d 表示从 CD-ROM 启动。
-vnc：启用 VNC 服务器，允许通过 VNC 客户端远程访问虚拟机的图形界面。
-spice：启用 SPICE 连接，类似于 VNC，但提供了更好的性能和特性。
-netdev 和 -device：配置虚拟机的网络设备。
-fsdev：挂载宿主机的文件系统到虚拟机。
-usb：启用 USB 支持。
-serial 和 -parallel：配置虚拟机的串行和并行端口。
-global：设置全局选项，例如启用或禁用特定的硬件特性。
-display：选择显示引擎，例如 -display sdl 或 -display gtk。
-kernel：直接启动一个 Linux 内核，而不是从 CD-ROM 或硬盘启动。
-append：指定启动时传递给操作系统的附加参数。
-args：允许你指定传递给 QEMU 应用程序的参数。
-pidfile：创建一个包含虚拟机进程 ID 的文件。
-snapshot：启用虚拟硬盘的快照支持。
-monitor：启用 QEMU 监控器，允许你与虚拟机交互。

### Build Python Wheel包
确保你的 LoongArch 架构的 Python 解释器已经安装了 setuptools、wheel 和构建 wheel 包所需的其他工具。
bash
python3 -m pip install setuptools wheel
Python3 setup.py bdist_wheel

得到 ncnn-x.x.x+xx.xxxxxxxx-py3-none-linux_loongarch64.whl
Linux_loongarch64说明支持LoongArch架构，.whl说明这是一个wheel包。

### 将生成的wheel包使用交叉编译的Python解释器安装
bash
python3 -m pip install dist/*.whl

### 测试vulkan是否正常工作
需要编写简单的Vulkan 测试代码

### 注意事项
每个/path/to/your/xx都需要替换成你相应的文件路径。
确保交叉编译工具链中的编译器、链接器等工具能够正确地为目标架构生成代码。
确保所有依赖项都适用于LoongArch架构，并且已经正确安装。
如果在编译过程中遇到问题，检查错误信息，并根据需要调整编译选项或环境变量。

由于我QEMU虚拟机启动有问题所以后续测试并不清楚具体操作。。
