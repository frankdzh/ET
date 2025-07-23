# CLAUDE.md

此文件为Claude Code (claude.ai/code) 在此代码库中工作时提供指导。

## 项目概述

ET (Entity-Task) 是一个Unity3D + .NET Core分布式游戏框架，支持客户端和服务端双端开发。这是8.1版本，采用多线程/多进程架构，基于类似Erlang进程的纤程(Fiber)设计。

## 核心架构组件

### 程序集结构
- **Unity.Core**: 客户端/服务端共享的核心功能（Entity系统、网络、序列化）
- **Unity.Model**: 数据模型和组件（支持热重载）
- **Unity.Hotfix**: 热重载游戏逻辑（客户端和服务端）
- **Unity.Loader**: 启动引导和资源加载
- **DotNet.App**: 独立服务器应用程序入口点
- **DotNet.Model/Core/Hotfix**: 服务端程序集

### 核心系统
- **Entity-Component系统**: 所有游戏对象继承自Entity，通过Component添加功能
- **纤程系统**: 轻量级线程，利用多核同时保持单线程开发体验
- **Actor模型**: 进程间实体的位置透明消息传递
- **热重载**: 客户端和服务端运行时代码重载，无需重启
- **网络层**: 支持TCP、UDP、KCP、WebSocket并具有自动回退功能

## 开发命令

### Unity编辑器命令（F键）
- **F6**: 编译热重载程序集（`ET -> Compile`）
- **F7**: 运行时热重载代码（`ET -> Reload`）

### 构建和工具
```bash
# 服务端编译（.NET）
dotnet build ET.sln

# 导出Excel配置为代码
ET -> BuildTool -> ExcelExporter

# 生成protobuf C#文件
ET -> BuildTool -> Proto2CS

# 启动独立服务器
ET -> ServerTools -> Start Server(Single Process)
```

### 构建流程
1. 设置GlobalConfig：AppType（Demo/LockStep），CodeMode（Client/Server/ClientServer）
2. 使用F6编译或在IDE中构建ET.sln
3. 打包：HybridCLR -> Generate -> All，然后BuildTool -> BuildPackage

## 配置系统

### 环境配置
- **Localhost**: 本地开发（单机）
- **Release**: 生产部署
- **Benchmark**: 性能测试
- **RouterTest**: 软路由测试

### 关键配置文件
- `Assets/Resources/GlobalConfig`: Unity编辑器设置
- `Config/Json/cs/StartConfig/`: 服务器启动配置
- `Config/Excel/`: 游戏数据配置（转换为代码）

## 热重载系统

### 前提条件
Unity Preferences -> General -> 'ScriptChangesWhilePlaying' = 'RecompileAfterFinishedPlaying'

### 工作流程
1. 在游戏运行时修改代码
2. 按F6编译
3. 按F7重载 - 更改立即生效，无需重启

## 多模式支持

### 代码模式
- **Client**: 仅Unity客户端
- **Server**: 专用服务器
- **ClientServer**: 集成客户端+服务端（用于开发）

### 应用类型
- **Demo**: 状态同步演示
- **LockStep**: 预测回滚的帧同步

## 网络架构

### 协议
- **KCP**: 超快UDP可靠传输（游戏主要协议）
- **TCP**: 标准TCP，WebSocket回退
- **软路由**: 防DDoS保护层

### 消息系统
- **IMessage**: 所有网络消息的基础接口
- **Actor消息**: 位置透明的实体消息传递
- **RPC**: 异步/等待的请求-响应模式

## 测试和调试

### 机器人框架
机器人共享客户端逻辑代码，只需最少的额外代码即可进行大规模压力测试。

### 服务器工具
- **单进程模式**: 所有服务器在一个进程中用于调试
- **Watcher**: 进程监控和自动重启
- **REPL**: 运行时代码执行用于调试

## 开发要求

### 版本
- **Unity**: 2022.3.15（需要确切版本）
- **.NET**: .NET 8
- **IDE**: JetBrains Rider 2023.3+（官方不支持VS）

### 设置注意事项
- 需要全局VPN下载Unity/NuGet包
- 服务器HTTP服务需要管理员权限
- 外部脚本编辑器：Rider（Generate .csproj files: 全部不勾选）

## 常见工作流程

### 添加新功能
1. 在Model程序集中定义数据模型
2. 在Hotfix程序集中实现逻辑
3. 在HotfixView中添加UI/表现（仅客户端）
4. 使用F6/F7进行迭代开发

### 服务器部署
```bash
# Linux部署
dotnet publish -r linux-x64 --no-self-contained -c Release
# 复制Bin/和Config/到目标服务器
```

### 调试多进程问题
- 开发时使用单进程模式
- 检查ET/Logs/目录的错误日志
- 使用REPL模式进行运行时检查

## 性能特征

### 基准测试
- 平均每秒20万消息吞吐量
- 单服务器支持15k+并发玩家（64核/128GB测试）
- MemoryPack序列化实现零GC网络

### 内存管理
- 高频对象的对象池
- MemoryPack零分配序列化
- 已销毁实体的自动清理