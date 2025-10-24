<div align="center">

# ☕ JM - JAR管理工具

### 简化JAR管理工具 - 专业的Java应用管理解决方案

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/gokeep-projects/jar-manager)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](https://github.com/gokeep-projects/jar-manager/blob/main/LICENSE)
[![Shell](https://img.shields.io/badge/shell-bash-green.svg)](https://www.gnu.org/software/bash/)
[![Platform](https://img.shields.io/badge/platform-linux%20%7C%20macos-lightgrey.svg)]()

<br>

## 🚀 一键安装

```bash
wget  https://raw.githubusercontent.com/gokeep-projects/jar-manager/main/jm | sudo bash ./jm install
```

</div>

---

## 📋 目录

- [项目概述](#-项目概述)
- [功能特性](#-功能特性)
- [快速开始](#-快速开始)
- [安装指南](#-安装指南)
- [使用指南](#-使用指南)
- [配置文件](#-配置文件)
- [命令参考](#-命令参考)
- [使用示例](#-使用示例)
- [状态显示](#-状态显示)
- [最佳实践](#-最佳实践)

---

## 🎯 项目概述

JM (JAR Manager) 是一个功能强大的Shell脚本工具，专为简化Java JAR应用程序的管理而设计。它提供了直观的命令行界面，支持应用的启动、停止、重启、状态监控和JDK管理等功能。

### 核心优势
- 🔄 支持多JDK版本管理
- 📊 实时监控应用状态
- 🛡️ 自动重启守护进程
- 📈 友好的表格化状态显示

---

## ✨ 功能特性

| 功能类别     | 具体特性                | 图标 |
| ------------ | ----------------------- | ---- |
| **应用管理** | 启动、停止、重启JAR应用 | 🚀    |
| **状态监控** | 实时表格化状态显示      | 📊    |
| **JDK管理**  | 多版本JDK支持           | ⚙️    |
| **守护进程** | 自动监控和重启          | 🛡️    |
| **智能排序** | 按内存/CPU/状态排序     | 📈    |
| **命令生成** | 生成nohup启动命令       | 📝    |

---

## 🚀 快速开始

### 一键安装（推荐）

```bash
wget  https://raw.githubusercontent.com/gokeep-projects/jar-manager/main/jm | sudo bash ./jm install
```

### 手动安装

```bash
# 下载脚本
wget https://raw.githubusercontent.com/gokeep-projects/jar-manager/main/jm
chmod +x jm

# 安装到系统目录
sudo ./jm install
```

### 验证安装

```bash
jm help
```

---

## 📦 安装指南

### 系统要求
- Linux 或 macOS 系统
- Bash 4.0+
- Java 运行环境（可选，用于运行JAR应用）
- root 权限（用于系统安装）

### 安装方式

#### 方式一：一键安装脚本（推荐）
```bash
wget https://raw.githubusercontent.com/gokeep-projects/jar-manager/main/jm | sudo bash ./jm install
```

#### 方式二：手动下载安装
```bash
# 下载脚本
wget https://raw.githubusercontent.com/gokeep-projects/jar-manager/main/jm
chmod +x jm

# 安装到系统目录
sudo ./jm install
```

#### 方式三：开发模式使用
```bash
# 直接下载使用
wget https://raw.githubusercontent.com/gokeep-projects/jar-manager/main/jm
chmod +x jm
./jm help
```

### 卸载方法
```bash
# 如果已安装到系统目录
sudo jm uninstall

# 如果是本地使用
rm -f jm
```

---

## 📖 使用指南

### 配置文件

JM 支持多级配置文件，按以下顺序搜索：

1. **当前目录**: `./jm.conf`
2. **脚本目录**: `$SCRIPT_DIR/jm.conf`
3. **系统目录**: `/etc/jm/jm.conf`

### 基本工作流程

#### 1. 创建配置文件

创建配置文件 `/etc/jm/jm.conf`：

```ini
# Web应用配置
[webapp]
path=/opt/apps/webapp.jar
args=-Xms256m -Xmx512m -Dspring.profiles.active=prod
jdk=/usr/lib/jvm/java-11-openjdk
daemon=true
startupTimeout=30

# API服务配置
[api-service]
path=/opt/apps/api.jar
args=-Xms1g -Xmx2g
daemon=true
```

#### 2. 启动应用

```bash
# 启动所有配置的应用
jm start

# 启动指定应用
jm start webapp
```

#### 3. 监控状态

```bash
# 查看所有应用状态
jm status

# 动态监控（每秒刷新）
jm status -f

# 按内存使用率排序
jm status -s -m

# 按CPU使用率排序并动态刷新
jm status -s -c -f
```

---

## ⚙️ 配置文件

### 配置文件格式

配置文件采用INI格式，每个应用一个配置段：

```ini
[应用名称]
path=JAR文件绝对路径 (必需)
jdk=JDK绝对路径 (可选)
args=JVM参数 (可选)
daemon=true|false (可选, 默认: false)
startupTimeout=30 (可选, 默认: 60秒)
```

### 配置参数说明

| 参数             | 描述                 | 是否必需 | 默认值      |
| ---------------- | -------------------- | -------- | ----------- |
| `path`           | JAR文件的绝对路径    | 是       | 无          |
| `jdk`            | JDK的绝对路径        | 否       | 系统默认JDK |
| `args`           | JVM启动参数          | 否       | 空          |
| `daemon`         | 是否启用守护进程模式 | 否       | false       |
| `startupTimeout` | 启动超时时间（秒）   | 否       | 60          |

### 配置示例

```ini
# 生产环境Web应用
[webapp]
path=/opt/myapp/myapp.jar
args=-Xms512m -Xmx1g -Dspring.profiles.active=prod
jdk=/usr/lib/jvm/jdk-17.0.10
daemon=true
startupTimeout=30

# 测试环境API服务
[api-test]
path=/opt/test/api-test.jar
args=-Xms256m -Xmx512m -Dspring.profiles.active=test
daemon=false

# 批处理作业
[batch-job]
path=/opt/batch/batch-processor.jar
args=-Xms1g -Xmx2g -XX:+UseG1GC
jdk=/usr/lib/jvm/jdk-8u281
```

---

## 📝 命令参考

### 应用管理命令

#### 启动应用
```bash
# 启动所有应用
jm start

# 启动指定应用
jm start [应用名称]

# 示例
jm start webapp
jm start api-service
```

#### 停止应用
```bash
# 停止所有应用
jm stop

# 停止指定应用
jm stop [应用名称]

# 示例
jm stop webapp
```

#### 重启应用
```bash
# 重启所有应用
jm restart

# 重启指定应用
jm restart [应用名称]

# 示例
jm restart webapp
```

### 状态监控命令

#### 基本状态查看
```bash
# 显示所有应用状态
jm status
```

#### 状态排序选项
```bash
# 按内存使用率排序（从高到低）
jm status -s -m

# 按CPU使用率排序（从高到低）
jm status -s -c

# 按状态排序（运行中优先）
jm status -s -t
```

#### 动态监控
```bash
# 每秒刷新状态
jm status -f

# 按内存排序并动态刷新
jm status -s -m -f

# 按CPU排序并动态刷新
jm status -s -c -f
```

### 其他命令

#### 生成启动命令
```bash
# 生成所有应用的nohup命令
jm gen

# 生成指定应用的nohup命令
jm gen [应用名称]

# 示例
jm gen webapp
```

#### 查看JDK信息
```bash
# 查看系统JDK信息
jm jdk

# 查看指定应用的JDK信息
jm jdk [应用名称]

# 示例
jm jdk webapp
```

#### 系统管理
```bash
# 安装JM到系统目录
jm install

# 卸载JM
jm uninstall

# 进入守护进程模式
jm daemon
```

#### 帮助信息
```bash
# 显示帮助信息
jm help
```

---

## 💡 使用示例

### 完整工作流程

#### 1. 创建配置文件

```bash
# 创建配置目录
sudo mkdir -p /etc/jm

# 创建配置文件
sudo nano /etc/jm/jm.conf
```

配置文件内容：
```ini
[webapp]
path=/opt/apps/webapp.jar
args=-Xms256m -Xmx512m -Dspring.profiles.active=prod
jdk=/usr/lib/jvm/java-11-openjdk
daemon=true
startupTimeout=30

[api-service]
path=/opt/apps/api.jar
args=-Xms1g -Xmx2g
daemon=true

[batch-job]
path=/opt/apps/batch.jar
args=-Xms512m -Xmx1g
```

#### 2. 启动应用

```bash
# 启动所有应用
jm start

# 输出示例
Starting webapp...
webapp started successfully with PID 12345
Starting api-service...
api-service started successfully with PID 12346
Starting batch-job...
batch-job started successfully with PID 12347
```

#### 3. 监控状态

```bash
# 查看状态
jm status

# 动态监控
jm status -f
```

#### 4. 生成启动命令

```bash
# 生成webapp的启动命令
jm gen webapp

# 输出示例
Application: webapp
Generated nohup command:
nohup /usr/lib/jvm/java-11-openjdk/bin/java -Xms256m -Xmx512m -Dspring.profiles.active=prod -jar "/opt/apps/webapp.jar" > "/dev/null" 2>&1 &
Log file location: /dev/null
```

#### 5. 停止应用

```bash
# 停止指定应用
jm stop webapp

# 停止所有应用
jm stop
```

---

## 📊 状态显示

JM 提供美观的表格化状态显示：

```
┌─────────────┬───────┬─────────────────────┬─────────────┬──────────┬─────────┬─────────┐
│ App Name    │ PID   │ JAR Path            │ Ports       │ Memory   │ CPU     │ Status  │
├─────────────┼───────┼─────────────────────┼─────────────┼──────────┼─────────┼─────────┤
│ webapp      │ 12345 │ /opt/apps/webapp.jar│ 8080,8081   │ 256.3M   │ 5.2%    │ Running │
│ api-service │ 12346 │ /opt/apps/api.jar   │ 9090        │ 512.1M   │ 12.8%   │ Running │
│ batch-job   │ --    │ /opt/apps/batch.jar │ --          │ --       │ --      │ Stopped │
└─────────────┴───────┴─────────────────────┴─────────────┴──────────┴─────────┴─────────┘
```

### 状态说明

- **🟢 Running** - 应用正在运行
- **🔴 Crashed** - 应用崩溃
- **⚫ Stopped** - 应用已停止
- **🟡 Unknown** - 状态未知

### 监控指标

- **PID** - 进程ID
- **Ports** - 监听的端口号
- **Memory** - 内存使用量
- **CPU** - CPU使用率
- **Start Time** - 启动时间

---

## 🎯 最佳实践

### 1. 生产环境配置

```ini
[production-app]
path=/opt/prod/myapp.jar
args=-Xms2g -Xmx4g -XX:+UseG1GC -Dspring.profiles.active=production
jdk=/usr/lib/jvm/jdk-11.0.14
daemon=true
startupTimeout=120
```

### 2. 测试环境配置

```ini
[test-app]
path=/opt/test/myapp.jar
args=-Xms512m -Xmx1g -Dspring.profiles.active=test
daemon=false
```

### 3. 监控建议

- 使用 `jm status -f` 进行实时监控
- 设置合理的内存限制避免OOM
- 为生产应用启用daemon模式
- 定期检查JDK版本一致性

### 4. 性能优化

- 合理设置JVM堆内存大小
- 使用G1GC垃圾收集器
- 配置适当的启动超时时间
- 监控CPU和内存使用率

---

## 🔧 故障排除

### 常见问题

#### 1. 应用启动失败
```bash
# 检查配置文件
jm status

# 查看生成的启动命令
jm gen [应用名称]

# 手动执行启动命令测试
```

#### 2. 找不到JDK
```bash
# 检查系统JDK
jm jdk

# 检查配置文件中的JDK路径
which java
echo $JAVA_HOME
```

#### 3. 权限问题
```bash
# 检查文件权限
ls -la /path/to/jar/file
ls -la /path/to/jdk/

# 修改权限
chmod +x /path/to/jar/file
```

#### 4. 端口冲突
```bash
# 检查端口占用
netstat -tulpn | grep [端口号]
ss -tulpn | grep [端口号]
```

---

## 📚 高级功能

### 守护进程模式

JM 支持守护进程模式，自动监控配置了 `daemon=true` 的应用：

```bash
# 启动守护进程
jm daemon

# 守护进程会自动重启异常停止的应用
```

### 批量操作

支持对多个应用进行批量操作：

```bash
# 启动所有应用
jm start

# 停止所有应用
jm stop

# 重启所有应用
jm restart
```

### 自定义排序

支持多种排序方式：

```bash
# 按内存排序
jm status -s -m

# 按CPU排序
jm status -s -c

# 按状态排序
jm status -s -t
```

---

## 🤝 贡献与支持

### 获取帮助

```bash
# 查看帮助信息
jm help

# 查看版本信息
jm --version
```

### 报告问题

如有问题或建议，请访问：
🌐 **[GitHub Issues](https://github.com/gokeep-projects/jar-manager/issues)**

### 贡献代码

欢迎提交 Pull Request：
🌐 **[GitHub Repository](https://github.com/gokeep-projects/jar-manager)**

---

## 📄 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情。

---

## 🙏 致谢

感谢所有贡献者和使用者的支持！

<div align="center">

**☕ JM - 让 JAR 应用管理变得更简单！**

[![Star History Chart](https://api.star-history.com/svg?repos=gokeep-projects/jar-manager&type=Date)](https://star-history.com/#gokeep-projects/jar-manager&Date)

</div>