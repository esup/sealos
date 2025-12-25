# Sealos Development Environment Setup Guide

This guide will help you set up a complete local development environment for the Sealos project.

## System Requirements

### Operating System
- **Linux**: Required (sealos binary requires CGO for overlay driver support, Linux only)
- **macOS/Windows**: Can develop using a Linux virtual machine (e.g., [multipass](https://multipass.run/))

### Software Dependencies

#### 1. Go Environment
- **Version**: Go 1.23 or higher
- **Installation**:
  ```bash
  wget https://go.dev/dl/go1.23.linux-amd64.tar.gz
  tar -C /usr/local -zxvf go1.23.linux-amd64.tar.gz
  cat >> /etc/profile <<EOF
  # set go path
  export PATH=\$PATH:/usr/local/go/bin
  EOF
  source /etc/profile && go version
  ```

#### 2. Node.js Environment (Frontend Development)
- **Version**: Node.js 20.4.0 or higher
- **Package Manager**: pnpm 8.9.0
- **Installation**:
  ```bash
  # Install Node.js (nvm recommended)
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
  source ~/.bashrc
  nvm install 20.4.0
  
  # Install pnpm
  npm install -g pnpm@8.9.0
  ```

#### 3. System Libraries and Tools
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

#### 4. Optional Tools
- **Docker**: For container image building and testing
- **golangci-lint**: Code linting tool (version 1.46.2 or higher)
  ```bash
  go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
  ```

## Repository Structure Overview

```
sealos/
├── lifecycle/          # Core CLI and binaries (sealos, sealctl, lvscare, etc.)
├── controllers/        # Kubernetes controllers (multiple independent Go modules)
├── frontend/           # Frontend applications (Next.js workspace with pnpm)
├── service/            # Backend services (independent Go modules)
├── webhooks/           # Kubernetes admission webhooks
├── docs/               # Documentation
└── scripts/            # Build and deployment scripts
```

## Clone the Repository

```bash
git clone https://github.com/labring/sealos.git
cd sealos
```

## Building the Project

### 1. Build Core sealos CLI (lifecycle directory)

```bash
cd lifecycle

# Build a single binary (e.g., sealos)
make build BINS=sealos

# Build all binaries
make build

# Build multi-architecture versions
make build.multiarch PLATFORMS="linux_amd64 linux_arm64"
```

**Build Artifacts**:
- Binaries are located in: `lifecycle/bin/linux_amd64/` or `lifecycle/bin/linux_arm64/`
- Main binaries: `sealos`, `sealctl`, `lvscare`, `image-cri-shim`

**Important Notes**:
- First build may take 3-4 minutes to download dependencies
- Subsequent builds typically take 10-15 seconds
- `sealos` binary must be built on Linux (requires CGO)

### 2. Build Controllers

Each controller is an independent Go module with its own Makefile:

```bash
# Example: Build the user controller
cd controllers/user
make build

# Build artifacts are in
ls bin/manager
```

### 3. Build Frontend Applications

```bash
cd frontend

# Install dependencies (first time)
pnpm install

# Build all packages
pnpm run build-packages

# Build all provider applications
pnpm run build-providers

# Run development server
pnpm dev-desktop        # Desktop app
pnpm dev-app           # App launchpad
pnpm dev-db            # Database provider
pnpm dev-terminal      # Terminal
# More commands in package.json
```

**Environment Variables**:
- Copy `.env.template` to `.env` and fill in configurations
- Generate Prisma clients for desktop app:
  ```bash
  cd desktop
  pnpm gen:global && pnpm gen:region
  ```

## Development Workflow

### Code Formatting

```bash
cd lifecycle
make format     # Format lifecycle code
```

### Code Linting

```bash
cd lifecycle
make lint       # Run linter
```

### Run Tests

```bash
cd lifecycle
go test ./pkg/...    # Run unit tests
```

### Validate Build

```bash
cd lifecycle
./bin/linux_amd64/sealos version      # Check version
./bin/linux_amd64/sealos help         # View help
./bin/linux_amd64/sealctl hostname    # Test sealctl
```

## Go Workspace Usage

Sealos uses Go 1.18+ workspace feature. When adding new modules:

```bash
# Run in the respective directory
go work use -r .
```

Each major directory (lifecycle, controllers, service, webhooks) has its own `go.work` file.

## Creating New CRD and Controller

```bash
# Set variables
export CRD_NAME=YourCRDName
export CRD_GROUP=yourgroup

# Create controller directory and initialize
mkdir controllers/${CRD_NAME}
cd controllers/${CRD_NAME}
kubebuilder init --domain sealos.io --repo github.com/labring/sealos/controllers/${CRD_NAME}

# Update workspace
go work use -r .

# Create API
kubebuilder create api --group ${CRD_GROUP} --version v1 --kind ${CRD_NAME}
```

## Developing on macOS with multipass (ARM64)

If you're using macOS (especially Apple Silicon), you can use multipass to create a Linux VM:

```bash
# Set your source code directory
export SEALOS_CODE_DIR=/path/to/your/sealos

# Launch VM and mount code
multipass launch \
  --mount ${SEALOS_CODE_DIR}:/go/src/github.com/labring/sealos \
  --name sealos-dev --cpus 2 --mem 4G --disk 40G

# Enter VM
multipass exec sealos-dev bash
sudo su

# Install dependencies in VM
apt-get update
apt-get install -y build-essential make pkg-config \
  libdevmapper-dev libbtrfs-dev libgpgme-dev

# Install Go (ARM64 version)
wget https://go.dev/dl/go1.23.linux-arm64.tar.gz
tar -C /usr/local -zxvf go1.23.linux-arm64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# Build project
cd /go/src/github.com/labring/sealos/lifecycle
make build
```

## Troubleshooting

### 1. Slow Git Clone
Use a proxy mirror:
```bash
git clone https://ghproxy.com/https://github.com/labring/sealos
```

### 2. Slow Go Package Downloads
Set GOPROXY:
```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

### 3. Compile Error: "C compiler not found"
Install gcc:
```bash
# Debian/Ubuntu
apt-get install build-essential

# CentOS/RHEL
yum install gcc gcc-c++
```

### 4. Frontend Dependency Installation Fails
Ensure you're using the correct Node.js and pnpm versions:
```bash
node --version  # Should be 20.4.0+
pnpm --version  # Should be 8.9.0
```

## Contributing Guidelines

Before submitting a Pull Request:

1. **Format code**: `make format`
2. **Run linter**: `make lint`
3. **Test your changes**: Run relevant tests
4. **Follow commit conventions**: Use [Conventional Commits](https://www.conventionalcommits.org/)
5. **Read contributing guide**: [CONTRIBUTING.md](./CONTRIBUTING.md)

## Additional Resources

- **Documentation**: https://sealos.io/docs
- **API Reference**: https://github.com/labring/sealos.io
- **Community Support**: 
  - Discord: https://discord.gg/qzBmGGZGk7
  - GitHub Issues: https://github.com/labring/sealos/issues

## Important Notes

- **Cross-platform builds**: All binaries except `sealos` can be built anywhere with `CGO_ENABLED=0`
- **Testing limitations**: Some sealos commands require Linux privileges (unshare, overlay mounts) and may not work in sandboxed environments
- **CI integration**: Ensure code passes all CI checks (formatting, license, build) before submitting

---

Questions? Please ask in [GitHub Issues](https://github.com/labring/sealos/issues) or join our [Discord community](https://discord.gg/qzBmGGZGk7).
