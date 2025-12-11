# DiskANN 环境配置 - Docker 方式（macOS 推荐）

本文档介绍如何在 macOS 上使用 Docker 运行和开发 DiskANN。

**为什么使用 Docker？**
- DiskANN 在 Linux 环境下测试最充分
- Docker 提供与 CI 环境一致的依赖和配置
- 无需在 macOS 上安装复杂的依赖（如 MKL、libaio 等）
- 开箱即用，避免环境配置问题

---

## 前置条件

✅ 已安装 Docker Desktop for Mac

如果还没安装，请从官网下载：https://docs.docker.com/desktop/mac/install/

验证 Docker 是否正常运行：
```bash
docker --version
docker ps
```

---

## 完整操作流程

### 步骤 1：进入 DiskANN 仓库目录

```bash
cd /Users/jack/Desktop/Intership/Code/DiskANN-learn
```

### 步骤 2：构建 Docker 开发镜像

这一步会根据仓库提供的 `DockerfileDev` 构建一个包含所有依赖的 Linux 开发环境。

```bash
docker build -f DockerfileDev -t diskann-dev:local .
```

**说明：**
- `-f DockerfileDev`：指定使用 DockerfileDev 文件
- `-t diskann-dev:local`：给镜像打标签，方便后续使用
- `.`：构建上下文为当前目录

⏱️ 首次构建需要 5-15 分钟（取决于网络速度），镜像会自动安装：
- CMake、g++、make 等编译工具
- Boost、MKL、libaio、gperftools 等依赖库
- Python 3.10 环境

### 步骤 3：启动 Docker 容器

```bash
docker run -it --name diskann-dev --rm \
  -v "$PWD":/workspace \
  -w /workspace \
  diskann-dev:local /bin/bash
```

**参数说明：**
- `-it`：交互式终端模式
- `--name diskann-dev`：容器名称
- `--rm`：退出容器后自动删除（下次重新创建）
- `-v "$PWD":/workspace`：挂载当前目录到容器的 /workspace（代码修改会实时同步）
- `-w /workspace`：设置工作目录为 /workspace
- `/bin/bash`：启动 bash shell

✅ 执行后你会进入容器内部，提示符变为类似 `root@xxxxx:/workspace#`

### 步骤 4：在容器内编译 DiskANN

现在你已经在 Linux 容器环境中了，可以开始编译：

```bash
# 初始化 git submodule（如果还没初始化）
git submodule init && git submodule update --recursive

# 创建 build 目录并进入
mkdir -p build && cd build

# 使用 CMake 配置项目（Release 模式）
cmake -DCMAKE_BUILD_TYPE=Release ..

# 编译（使用所有可用 CPU 核心）
make -j$(nproc)
```

⏱️ 编译时间约 3-10 分钟

编译成功后，所有可执行文件在 `build/apps/` 目录下。

### 步骤 5：验证编译结果

查看生成的可执行文件：

```bash
# 列出编译好的工具
ls -lh apps/

# 常用的工具包括：
# - build_memory_index：构建内存索引
# - search_memory_index：检索内存索引
# - build_disk_index：构建磁盘索引
# - search_disk_index：检索磁盘索引
```

查看某个工具的帮助信息：

```bash
./apps/build_memory_index --help
```

### 步骤 6：准备测试数据（可选）

如果你想快速测试，可以生成一个小规模的随机数据集：

```bash
# 在容器内创建测试数据目录
mkdir -p /workspace/test_data

# 使用 Python 生成随机向量数据
python3 << 'EOF'
import struct
import numpy as np

# 生成 1000 个 128 维的随机向量作为数据库
n_points = 1000
dim = 128
data = np.random.randn(n_points, dim).astype(np.float32)

# 保存为 DiskANN 格式（二进制）
with open('/workspace/test_data/random_vectors.bin', 'wb') as f:
    f.write(struct.pack('i', n_points))  # 数据点数量
    f.write(struct.pack('i', dim))        # 维度
    data.tofile(f)

# 生成 10 个查询向量
n_queries = 10
queries = np.random.randn(n_queries, dim).astype(np.float32)
with open('/workspace/test_data/query_vectors.bin', 'wb') as f:
    f.write(struct.pack('i', n_queries))
    f.write(struct.pack('i', dim))
    queries.tofile(f)

print(f"✅ 已生成测试数据：{n_points} 个向量，{n_queries} 个查询，维度 {dim}")
EOF
```

### 步骤 7：构建并检索索引（完整示例）

```bash
# 构建内存索引
./apps/build_memory_index \
  --data_type float \
  --dist_fn l2 \
  --data_path /workspace/test_data/random_vectors.bin \
  --index_path_prefix /workspace/test_data/index_random

# 执行检索
./apps/search_memory_index \
  --data_type float \
  --dist_fn l2 \
  --index_path_prefix /workspace/test_data/index_random \
  --query_file /workspace/test_data/query_vectors.bin \
  --K 10 \
  --result_path /workspace/test_data/search_results.txt

# 查看检索结果
head -20 /workspace/test_data/search_results.txt
```

**参数说明：**
- `--data_type float`：数据类型（float/int8/uint8）
- `--dist_fn l2`：距离度量（l2/cosine/mips）
- `--K 10`：返回前 10 个最近邻

---

## 常用操作

### 退出容器

```bash
exit
# 或按 Ctrl+D
```

由于使用了 `--rm` 参数，容器会自动删除，但编译结果保留在本地 `build/` 目录中。

### 重新进入开发环境

```bash
# 确保在仓库根目录
cd /Users/jack/Desktop/Intership/Code/DiskANN-learn

# 重新启动容器（使用相同命令）
docker run -it --name diskann-dev --rm \
  -v "$PWD":/workspace \
  -w /workspace \
  diskann-dev:local /bin/bash

# 进入 build 目录继续工作
cd build
```

### 在容器外查看日志或文件

由于挂载了本地目录，所有在容器内生成的文件都会同步到你的 macOS 目录：

```bash
# 在 macOS 终端中
ls -lh /Users/jack/Desktop/Intership/Code/DiskANN-learn/build/apps/
cat /Users/jack/Desktop/Intership/Code/DiskANN-learn/test_data/search_results.txt
```

### 清理 Docker 资源（可选）

如果需要重新构建镜像或清理空间：

```bash
# 删除镜像
docker rmi diskann-dev:local

# 清理未使用的镜像和容器
docker system prune -a
```

---

## 常见问题排查

### 1. Docker 启动失败

确保 Docker Desktop 正在运行：
```bash
open -a Docker
```

### 2. 构建镜像时网络超时

可以配置 Docker 使用国内镜像源，编辑 Docker Desktop → Preferences → Docker Engine，添加：
```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/"]
}
```

### 3. 磁盘空间不足

Docker 镜像较大（约 2-3GB），确保有足够空间：
```bash
docker system df
```

### 4. 容器内无法访问 GPU

DiskANN 主要使用 CPU。如需 GPU 支持，需要额外配置 NVIDIA Container Toolkit。

---

## 下一步学习

- 查看 `workflows/` 目录下的详细文档了解更多功能
- 尝试使用真实数据集（如 SIFT1M）测试性能
- 探索 Python 绑定：在容器内 `cd python && pip install -e .`

---

## 参考资料

- 项目主 README：`/Users/jack/Desktop/Intership/Code/DiskANN-learn/README.md`
- Dockerfile 定义：`DockerfileDev`
- 官方 Workflows：`workflows/*.md`

