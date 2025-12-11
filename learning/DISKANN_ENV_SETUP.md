 # DiskANN 环境配置（适用于 macOS）

 本文档给出在 macOS 平台上运行和开发 DiskANN 的推荐配置方法。我们提供两种可选方式：

 - 方式 A（推荐）：使用 Docker 在 Linux 容器中运行与开发，保证与 CI/测试环境的一致性。
 - 方式 B（macOS 原生）：在 macOS 上安装依赖并原生编译（可用，但某些 Linux 专属功能可能受限，比如 libaio）。

---

 ## 方式 A — Docker（macOS 推荐）

 为什么使用 Docker？DiskANN 在 Linux 与 Windows 上测试更全面。对于 macOS 用户来说，使用 Docker 可以获得与 CI 环境一致的镜像和依赖，最简单、最可复现。

 前置条件：
 - Docker Desktop for Mac（https://docs.docker.com/desktop/mac/install/）
 - 可选：如果打算用大型数据集，建议在 Docker 设置中分配更多内存和 CPU。

Steps:

 1. 构建开发镜像（快速且可复现）：

```bash
# From the repo root
# Build the development image defined by DockerfileDev
docker build -f DockerfileDev -t diskann-dev:local .
```

 2. 启动容器并挂载仓库以便迭代开发：

```bash
docker run -it --name diskann-dev --rm -v "$PWD":/workspace -w /workspace diskann-dev:local /bin/bash
```

 容器内快速构建示例：

```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
```

 3. 运行示例（内存索引 / 检索）：

```bash
# Replace with the actual binary path; built binaries are under apps or src output
./apps/search_memory_index -db_file <path-to-dataset> -query_file <path-to-queries> -indexType in-memory
```

 更多用法请参见仓库根目录下的 `/workflows/*` 文档。

---

 ## 方式 B — macOS 原生构建

 注意：macOS 不是 DiskANN 官方常测平台，原生构建在核心功能上一般可用，但某些基于 Linux 的特性（例如 `libaio` 或异步文件读写）可能不可用或需要额外适配。

 ### 1） 前置依赖

 - Xcode Command Line Tools（用于编译）
 - Homebrew（https://brew.sh/）
 - 使用 brew 安装常用依赖：

```bash
# Install Homebrew first if you don't have it
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install packages
brew update
brew install cmake boost gperftools openmp wget clang-format python3
```

 注意：macOS 上没有 `libaio`，DiskANN 在 Linux 上使用 `libaio` 做异步 IO。大多数内存索引功能仍可工作，但基于磁盘的异步索引功能可能受限。

 ### 2） 安装 Intel oneAPI MKL（推荐）

 DiskANN 在一些线性代数运算中依赖 MKL，因此推荐在 macOS 上安装 Intel oneAPI MKL，使得 CMake 能在默认路径下找到 MKL。

1. Download and install Intel oneAPI Base Toolkit and MKL (https://software.intel.com/content/www/us/en/develop/tools/oneapi/base-toolkit.html)
 2. 安装后执行 setvars 脚本，让编译器与 CMake 能找到 MKL 和 Intel OMP。例如安装在 `/opt/intel/oneapi` 后：

 ```bash
 source /opt/intel/oneapi/setvars.sh
 ```

 3. 如果 CMake 无法自动找到 MKL，可显式指定路径：

```bash
cmake -DMKL_PATH=/opt/intel/oneapi/mkl/latest -DMKL_INCLUDE_PATH=/opt/intel/oneapi/mkl/latest/include -DOMP_PATH=/opt/intel/oneapi/compiler/latest/linux/compiler/lib/intel64_lin -DCMAKE_BUILD_TYPE=Release ..
```

 根据实际安装路径调整上述参数。

 如果无法在 macOS 上使用 MKL，可以尝试 OpenBLAS 作为替代，但需要修改包含头文件或添加适配层，可能会有额外工作量。

 ### 3） 原生编译步骤

```bash
 ```bash
 # 在仓库根目录
 git submodule init && git submodule update --recursive
 mkdir -p build && cd build
 # 如有需要，提供 MKL 路径并确保 OpenMP 可用
 cmake -DCMAKE_BUILD_TYPE=Release -DMKL_PATH=/opt/intel/oneapi/mkl/latest -DMKL_INCLUDE_PATH=/opt/intel/oneapi/mkl/latest/include ..
 make -j$(sysctl -n hw.ncpu)
 ```
```

 可能注意事项 / 替代配置：
 - 如果 brew 安装的 OpenMP 无法被 clang 检测到，可通过 `brew install libomp` 并设置 `LDFLAGS` / `CPPFLAGS` 或 CMake 参数来指向 OpenMP 的库与头文件。
 - `libaio` 为 Linux 专属库，部分基于磁盘的异步功能会在 macOS 上不可用。
 - 若出现 MKL 相关错误，请检查 `MKL_PATH` 与 `MKL_INCLUDE_PATH` 是否正确。

 ### 4） 运行示例

```bash
# Run a built app, adjust binary path if needed
./apps/search_memory_index -db_file <path-to-dataset> -query_file <path-to-query> -indexType in-memory
```

 如果没有现成数据，可用 Python 生成小规模随机数据，然后用 DiskANN 的 `apps` 工具构建索引并测试查询。

---

 ## Python 绑定（可选）

 仓库包含 `python/` 子模块和 `diskannpy`，可用于从 Python 调用 DiskANN。若想使用 Python 绑定，请进入 `python` 并按 `python/README.md` 中的指引操作：

```bash
cd python
# Typically, it relies on building the C++ project and then installing diskannpy
pip install -e .
```

---

 ## 常见问题排查（Troubleshooting）

 - 如果 `cmake` 报错找不到 `Boost`：使用 `brew install boost` 安装，然后运行 `cmake` 时添加 `-DBOOST_ROOT=$(brew --prefix boost)`。
 - 如果遇到 MKL 相关错误：验证 `MKL_PATH` / `MKL_INCLUDE_PATH` 是否正确，并确认已 `source /opt/intel/oneapi/setvars.sh`。
 - 如果你不想安装 MKL，推荐使用 Docker 方案。

---

 ## 下一步：示例与学习笔记

 我可以为你在 `learning/jack/` 下添加一个完整示例（下载小数据集、构建内存索引并运行检索），并添加一个中文 `notes.md`，详细说明 `apps` 中关键工具和命令如何使用。

---

References:
- Project README.md — top-level build instructions and links to `workflows/*.md`
- DockerfileDev — for reproducible dev environment
- Intel oneAPI (MKL) — https://www.intel.com/content/www/us/en/developer/tools/oneapi.html

