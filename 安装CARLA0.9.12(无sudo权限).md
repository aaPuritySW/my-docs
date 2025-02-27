# 安装CARLA0.9.12(无sudo权限)

## 1. 直接使用wget安装

找到carla官方的release页面: https://github.com/carla-simulator/carla/releases/tag/0.9.12/

我已经在我的目录下新建了`carla_0.9.12`文件夹, 找到对应版本的压缩包(比如我使用的是Linux的服务器我就wget tar.gz的文件, win就下载.zip)

```bash
cd ~/你的工作目录/carla_0.9.12
wget https://github.com/carla-simulator/carla/releases/download/0.9.12/CARLA_0.9.12.tar.gz
```

下载并且解压后我们需要检查文件是否存在, 以及是否文件类型正确:

```bash
ls -lh CARLA_0.9.12.tar.gz
file /home/工作目录/carla_0.9.12/carla-0-9-12-linux
```

第二行的输出包含gzip compressed data, ..., 说明它是**gzip压缩文件**，需要解压

```bash
mv home/你的工作目录/carla_0.9.12/carla-0-9-12-linux home/你的工作目录/carla_0.9.12/carla-0-9-12-linux.tar.gz
tar -xvzf /home/工作目录/carla_0.9.12/carla-0-9-12-linux.tar.gz -C /home/工作目录/carla_0.9.12/
```

解压后进入对应文件夹

```bash
cd /home/工作目录/carla_0.9.12/
ls
```

# 2. 安装**LLVM OpenMP 库**（与 OpenMPI 无关）

error while loading shared libraries: libomp.so.5, 你需要从github上下载软件https://github.com/llvm/llvm-project

```bash
git clone https://github.com/llvm/llvm-project.git
cd llvm-project/openmp
mkdir build && cd build
```

### 2.1 安装cmake

安装`camke`工具, 由于没有 `sudo` 权限，尝试使用 **conda** 安装

```bash
conda install cmake
```

验证安装：

```bash
cmake --version
```

如果安装成功，会显示 `cmake` 的版本信息

安装 `cmake` 后，重新进入 OpenMPI 的 `build` 目录，并运行以下命令：

```bash
cd ~/工作目录/OpenMP/openmpi-5.0.7/build
cmake ..
```

这应该会生成 Makefile，接下来就可以使用 `make` 编译了

### 2.2 安装 `autoconf`、`automake`、`libtool`

`CMakeLists.txt` **文件缺失**, `Makefile.am`、`Makefile.in` 和 `Makefile.ompi-rules` 是与 OpenMPI 构建相关的文件，但它们通常是自动生成 `Makefile` 的模板，而不是直接用于构建的 `Makefile` 文件。因此，你需要使用 `autoreconf` 和 `./configure` 来生成正确的 `Makefile`，然后再执行构建

```bash
conda install autoconf automake libtool
```

### 2.3 显式指定cmake安装路径

CMake 的安装路径未正确生效, 清理旧构建并重新配置

```bash
cd ~/工作目录/llvm-project/openmp

# 删除旧的 build 目录
rm -rf build

# 重新创建并进入 build 目录
mkdir build && cd build

# 显式指定安装路径（使用绝对路径）
cmake -DCMAKE_INSTALL_PREFIX=/home/工作目录/lib/libomp \
      -DLIBOMP_INSTALL_LIBDIR=/home/工作目录/lib/libomp/lib \
      ..

# 编译并安装
make -j$(nproc)
make install
```

验证安装路径是否成功安装到用户目录

```bash
ls /home/工作目录/lib/libomp/lib/libomp.so*
```

输出应有类似：

```markdown
libomp.so  libomp.so.5
```

如果已有 libomp.so 但 Carla 需要 libomp.so.5，可尝试强制创建符号链接：

**手动创建符号链接（解决版本不匹配）**

```bash
cd /home/工作目录/lib/libomp/lib
ln -sf libomp.so libomp.so.5
```

# 3. 更新OpenMP的环境变量(永久)

- **编辑** `~/.bashrc` **文件**：

```bash
nano ~/.bashrc
```

- **添加** `LD_LIBRARY_PATH` **设置**： 在文件的末尾添加以下内容：

```bash
export LD_LIBRARY_PATH="/home/工作目录/lib/libomp/lib:$LD_LIBRARY_PATH"
```

- **保存并退出**：
- 按 **Ctrl + X** 保存并退出。
- 按 **Y** 确认保存。

- **使改动生效**： 运行以下命令，重新加载 `~/.bashrc` 文件：

```bash
source ~/.bashrc
```

## **验证设置是否生效**

1. **检查** `LD_LIBRARY_PATH` **是否设置正确**：

```bash
echo $LD_LIBRARY_PATH
```

如果正确设置，你应该会看到类似下面的输出：

```bash
/home/your_user/lib/libomp/lib
```

2. 运行 Carla 时显式指定库路径, 观察输出中是否成功加载 `libomp.so.5`：

```bash
LD_DEBUG=libs ./CarlaUE4.sh -RenderOffScreen 2>&1 | grep libomp
```

3. **检查 libomp.so.5 是否可被找到**

```bash
ldconfig -p | grep libomp.so.5
```

​	如果返回空，手动确认路径

```bashls ~/local/libomp/lib/libomp.so.5
ls ~/local/libomp/lib/libomp.so.5
```

# 4. 运行CARLA

如果你在 Conda 环境中运行 Carla，尝试 **退出 Conda 环境**：

```bash
cd ~/工作目录/carla_0.9.12
conda deactivate
./CarlaUE4.sh -RenderOffScreen
```

### 验证 Carla 是否真正运行(在另外一个terminal运行)

#### (1) 检查进程是否存在

运行以下命令查看 Carla 进程是否在后台运行：

```bash
ps aux | grep CarlaUE4
```

- 如果看到类似 `/CarlaUE4-Linux-Shipping` 的进程，说明 Carla 正在运行。
- 如果进程不存在，说明启动失败。

#### (2) 检查端口监听状态

Carla 默认会监听端口 `2000` 和 `2001`。运行以下命令检查端口是否被占用：

```bash
netstat -tuln | grep '2000\|2001'
```

- 如果看到 `0.0.0.0:2000` 或 `:::2000`，说明 Carla 服务器已启动

### 关闭carla

- 在启动 Carla 的终端中按 `Ctrl+C`。
- 如果终端无响应，使用命令强制终止：

```bash
pkill -f CarlaUE4-Linux-Shipping
```

- **下一步操作**：通过 Python API 或 Carla 客户端工具进一步交互

------



## 同时需要注意一点, 你需要安装的是OpenMP而不是OpenMPI, 以下是OpenMPI的安装步骤

 官网: https://www.open-mpi.org/software/ompi/v5.0/, 找到对应版本下载, 如果没有版本信息则下载最新版

```bash
cd OpenMPI
wget https://download.open-mpi.org/release/open-mpi/v5.0/openmpi-5.0.7.tar.gz
tar -xvzf openmpi-5.0.7.tar.gz
cd openmpi-5.0.7
```

 然后创建一个 `build` 目录用于编译，并执行编译命令：  

```bash
mkdir build
cd build
```

 运行 `cmake` 来配置 OpenMPI

```bash
../configure --prefix=$HOME/lib/openmpi
```

```bash
cmake ..
```

1. ### **编译和安装 OpenMPI**： 

#### 1.1 **运行** `autoreconf` **和** `./configure`

在 `~/工作目录/OpenMP/openmpi-5.0.7` 目录下，运行以下命令：

##### 1.1.1**生成** `Makefile` **文件**：

```bash
autoreconf -i
```

##### 1.1.2**运行** `./configure` **配置 OpenMPI**：

 这个命令会检查你的系统并为构建生成必要的配置文件。你可以通过 `--prefix` 选项指定安装目录（如果你没有 `sudo` 权限，可以将它安装到用户目录）。

```basic
./configure --prefix=$HOME/lib/openmpi
```

这将会检查并配置 OpenMPI 的安装。

#### 1.2**编译和安装 OpenMPI**

一旦 `./configure` 成功运行并生成了 `Makefile`，你可以开始编译并安装 OpenMPI：

##### 1.2.1**编译 OpenMPI**：

```bash
make -j$(nproc)
```

这个命令会使用你的 CPU 核心并行编译 OpenMPI，速度会更快。

##### 1.2.2**安装 OpenMPI 到用户目录**：

 如果你使用了 `--prefix=$HOME/lib/openmpi` 选项配置，接下来你可以运行：

```bash
make install
```

这会将 OpenMPI 安装到你指定的目录（例如 `~/lib/openmpi`）。

2. ### **设置环境变量**：

    安装后，你需要设置环境变量 `LD_LIBRARY_PATH`，确保系统能够找到  

- **编辑** `~/.bashrc` **文件**：

```bash
nano ~/.bashrc
```

- **添加** `LD_LIBRARY_PATH` **设置**： 在文件的末尾添加以下内容：

```bash
export LD_LIBRARY_PATH=~/lib/openmpi/lib:$LD_LIBRARY_PATH
```

- **保存并退出**：
- 按 **Ctrl + X** 保存并退出。
- 按 **Y** 确认保存。

- **使改动生效**： 运行以下命令，重新加载 `~/.bashrc` 文件：

```bash
source ~/.bashrc
```

**验证设置是否生效**

1. **检查** `LD_LIBRARY_PATH` **是否设置正确**：

```bash
echo $LD_LIBRARY_PATH
```

如果正确设置，你应该会看到类似下面的输出：

```bash
/home/your_user/lib/openmpi/lib:/usr/lib:/lib...
```