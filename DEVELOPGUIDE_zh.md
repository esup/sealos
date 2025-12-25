# Sealos 开发环境搭建指南

本指南将帮助您在本地搭建 Sealos 项目的完整开发环境。

## 系统要求

### 操作系统
- **Linux**: 必需（sealos 二进制文件需要 CGO 支持 overlay 驱动，仅支持 Linux）
- **macOS/Windows**: 可以通过虚拟机（如 [multipass](https://multipass.run/)）运行 Linux 进行开发

### 软件依赖

#### 1. Go 环境
- **版本**: Go 1.23 或更高版本
- **安装方法**:
  ```bash
  wget https://go.dev/dl/go1.23.linux-amd64.tar.gz
  tar -C /usr/local -zxvf go1.23.linux-amd64.tar.gz
  cat >> /etc/profile <<EOF
  # set go path
  export PATH=\$PATH:/usr/local/go/bin
  EOF
  source /etc/profile && go version
  ```

#### 2. Node.js 环境（前端开发）
- **版本**: Node.js 20.4.0 或更高版本
- **包管理器**: pnpm 8.9.0
- **安装方法**:
  ```bash
  # 安装 Node.js（使用 nvm 推荐）
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
  source ~/.bashrc
  nvm install 20.4.0
  
  # 安装 pnpm
  npm install -g pnpm@8.9.0
  ```

#### 3. 系统库和工具
```bash
# Debian/Ubuntu
sudo apt-get update
sudo apt-get install -y \
  pkg-config \
  libdevmapper-dev \
  libbtrfs-dev \
  libgpgme-dev \
  build-essential \
  make

# CentOS/RHEL
sudo yum install -y \
  pkg-config \
  device-mapper-devel \
  btrfs-progs-devel \
  gpgme-devel \
  gcc \
  gcc-c++ \
  make
```

#### 4. 可选工具
- **Docker**: 用于容器镜像构建和测试
- **golangci-lint**: 代码检查工具（版本 1.46.2 或更高）
  ```bash
  go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
  ```

## 仓库结构概览

```
sealos/
├── lifecycle/          # 核心 CLI 和二进制文件（sealos, sealctl, lvscare 等）
├── controllers/        # Kubernetes 控制器（多个独立的 Go 模块）
├── frontend/           # 前端应用（Next.js 工作区，使用 pnpm）
├── service/            # 后端服务（独立的 Go 模块）
├── webhooks/           # Kubernetes 准入 Webhook
├── docs/               # 文档
└── scripts/            # 构建和部署脚本
```

## 克隆仓库

```bash
git clone https://github.com/labring/sealos.git
cd sealos
```

## 构建项目

### 1. 构建核心 sealos CLI（lifecycle 目录）

```bash
cd lifecycle

# 构建单个二进制文件（如 sealos）
make build BINS=sealos

# 构建所有二进制文件
make build

# 构建多架构版本
make build.multiarch PLATFORMS="linux_amd64 linux_arm64"
```

**构建产物**:
- 二进制文件位于: `lifecycle/bin/linux_amd64/` 或 `lifecycle/bin/linux_arm64/`
- 主要二进制文件: `sealos`, `sealctl`, `lvscare`, `image-cri-shim`

**注意事项**:
- 首次构建可能需要 3-4 分钟下载依赖
- 后续构建通常只需 10-15 秒
- `sealos` 二进制文件必须在 Linux 上构建（需要 CGO）

### 2. 构建控制器

每个控制器都是独立的 Go 模块，有自己的 Makefile：

```bash
# 示例：构建 user 控制器
cd controllers/user
make build

# 构建产物位于
ls bin/manager
```

### 3. 构建前端应用

```bash
cd frontend

# 安装依赖（首次运行）
pnpm install

# 构建所有包
pnpm run build-packages

# 构建所有 provider 应用
pnpm run build-providers

# 运行开发服务器
pnpm dev-desktop        # 桌面应用
pnpm dev-app           # 应用启动器
pnpm dev-db            # 数据库管理器
pnpm dev-terminal      # 终端
# 更多命令见 package.json
```

**环境变量**:
- 复制 `.env.template` 为 `.env` 并填写配置
- 为 desktop 应用生成 Prisma 客户端:
  ```bash
  cd desktop
  pnpm gen:global && pnpm gen:region
  ```

## 开发工作流

### 代码格式化

```bash
cd lifecycle
make format     # 格式化 lifecycle 代码
```

### 代码检查

```bash
cd lifecycle
make lint       # 运行 linter
```

### 运行测试

```bash
cd lifecycle
go test ./pkg/...    # 运行单元测试
```

### 验证构建

```bash
cd lifecycle
./bin/linux_amd64/sealos version      # 检查版本
./bin/linux_amd64/sealos help         # 查看帮助
./bin/linux_amd64/sealctl hostname    # 测试 sealctl
```

## Go Workspace 使用

Sealos 使用 Go 1.18+ 的工作区特性。添加新模块时：

```bash
# 在相应目录运行
go work use -r .
```

每个主要目录（lifecycle、controllers、service、webhooks）都有自己的 `go.work` 文件。

## 创建新的 CRD 和控制器

```bash
# 设置变量
export CRD_NAME=YourCRDName
export CRD_GROUP=yourgroup

# 创建控制器目录并初始化
mkdir controllers/${CRD_NAME}
cd controllers/${CRD_NAME}
kubebuilder init --domain sealos.io --repo github.com/labring/sealos/controllers/${CRD_NAME}

# 更新工作区
go work use -r .

# 创建 API
kubebuilder create api --group ${CRD_GROUP} --version v1 --kind ${CRD_NAME}
```

## 在 macOS 上使用 multipass 开发（ARM64）

如果您使用 macOS（特别是 Apple Silicon），可以使用 multipass 创建 Linux 虚拟机：

```bash
# 设置源代码目录
export SEALOS_CODE_DIR=/path/to/your/sealos

# 启动 VM 并挂载代码
multipass launch \
  --mount ${SEALOS_CODE_DIR}:/go/src/github.com/labring/sealos \
  --name sealos-dev --cpus 2 --mem 4G --disk 40G

# 进入 VM
multipass exec sealos-dev bash
sudo su

# 在 VM 中安装依赖
apt-get update
apt-get install -y build-essential make pkg-config \
  libdevmapper-dev libbtrfs-dev libgpgme-dev

# 安装 Go（ARM64 版本）
wget https://go.dev/dl/go1.23.linux-arm64.tar.gz
tar -C /usr/local -zxvf go1.23.linux-arm64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# 构建项目
cd /go/src/github.com/labring/sealos/lifecycle
make build
```

## 常见问题

### 1. 克隆代码慢
使用镜像代理:
```bash
git clone https://ghproxy.com/https://github.com/labring/sealos
```

### 2. 下载 Go 包慢
设置 GOPROXY:
```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

### 3. 编译错误: "C compiler not found"
安装 gcc:
```bash
# Debian/Ubuntu
apt-get install build-essential

# CentOS/RHEL
yum install gcc gcc-c++
```

### 4. 前端依赖安装失败
确保使用正确的 Node.js 和 pnpm 版本:
```bash
node --version  # 应该是 20.4.0+
pnpm --version  # 应该是 8.9.0
```

## 贡献指南

在提交 Pull Request 之前：

1. **格式化代码**: `make format`
2. **运行 linter**: `make lint`
3. **测试您的更改**: 运行相关测试
4. **遵循提交规范**: 使用 [Conventional Commits](https://www.conventionalcommits.org/zh-hans/)
5. **阅读贡献指南**: [CONTRIBUTING.md](./CONTRIBUTING.md)

## 更多资源

- **详细文档**: https://sealos.run/docs
- **API 参考**: https://github.com/labring/sealos.io
- **社区支持**: 
  - Discord: https://discord.gg/qzBmGGZGk7
  - GitHub Issues: https://github.com/labring/sealos/issues

## 注意事项

- **跨平台构建**: 除 `sealos` 外的所有二进制文件都可以在任何平台上使用 `CGO_ENABLED=0` 构建
- **测试限制**: 某些 sealos 命令需要 Linux 特权（unshare、overlay 挂载），在沙盒环境中可能无法运行
- **CI 集成**: 提交前确保代码通过所有 CI 检查（格式、许可证、构建）

---

有问题？请在 [GitHub Issues](https://github.com/labring/sealos/issues) 中提问或加入我们的 [Discord 社区](https://discord.gg/qzBmGGZGk7)。
