# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ET (Entity-Task) is a Unity3D + .NET Core distributed game framework that supports both client and server development. This is release 8.1, featuring multi-threaded/multi-process architecture with fiber-based design similar to Erlang processes.

## Key Architecture Components

### Assembly Structure
- **Unity.Core**: Shared core functionality between client/server (Entity system, networking, serialization)
- **Unity.Model**: Data models and components (hot-reloadable)
- **Unity.Hotfix**: Hot-reloadable game logic (client and server)
- **Unity.Loader**: Bootstrap and resource loading
- **DotNet.App**: Standalone server application entry point
- **DotNet.Model/Core/Hotfix**: Server-side assemblies

### Core Systems
- **Entity-Component System**: All game objects inherit from Entity, functionality added via Components
- **Fiber System**: Lightweight threads for multi-core utilization while maintaining single-threaded development experience
- **Actor Model**: Location-transparent messaging between entities across processes
- **Hot Reload**: Runtime code reloading for both client and server without restart
- **Network Layer**: Supports TCP, UDP, KCP, WebSocket with automatic fallback

## Development Commands

### Unity Editor Commands (F-keys)
- **F6**: Compile hot-reload assemblies (`ET -> Compile`)
- **F7**: Hot reload code at runtime (`ET -> Reload`)

### Build and Tools
```bash
# Server compilation (.NET)
dotnet build ET.sln

# Export Excel configs to code
ET -> BuildTool -> ExcelExporter

# Generate protobuf C# files  
ET -> BuildTool -> Proto2CS

# Start standalone server
ET -> ServerTools -> Start Server(Single Process)
```

### Build Process
1. Set GlobalConfig: AppType (Demo/LockStep), CodeMode (Client/Server/ClientServer)
2. Compile with F6 or build ET.sln in IDE
3. For packaging: HybridCLR -> Generate -> All, then BuildTool -> BuildPackage

## Configuration System

### Environment Configs
- **Localhost**: Local development (single machine)
- **Release**: Production deployment  
- **Benchmark**: Performance testing
- **RouterTest**: Soft routing testing

### Key Config Files
- `Assets/Resources/GlobalConfig`: Unity editor settings
- `Config/Json/cs/StartConfig/`: Server startup configurations
- `Config/Excel/`: Game data configurations (converted to code)

## Hot Reload System

### Prerequisites
Unity Preferences -> General -> 'ScriptChangesWhilePlaying' = 'RecompileAfterFinishedPlaying'

### Workflow
1. Make code changes while game is running
2. Press F6 to compile
3. Press F7 to reload - changes apply immediately without restart

## Multi-Mode Support

### Code Modes
- **Client**: Unity client only
- **Server**: Dedicated server  
- **ClientServer**: Integrated client+server for development

### App Types
- **Demo**: State synchronization demo
- **LockStep**: Frame synchronization with prediction rollback

## Network Architecture

### Protocols
- **KCP**: Ultra-fast UDP with reliability (primary for games)
- **TCP**: Standard TCP with WebSocket fallback
- **Soft Router**: Anti-DDoS protection layer

### Message System
- **IMessage**: Base interface for all network messages
- **Actor Messages**: Location-transparent entity messaging
- **RPC**: Request-response pattern with async/await

## Testing and Debugging

### Robot Framework
Robots share client logic code, enabling massive stress testing with minimal additional code.

### Server Tools
- **Single Process Mode**: All servers in one process for debugging
- **Watcher**: Process monitoring and automatic restart
- **REPL**: Runtime code execution for debugging

## Development Requirements

### Versions
- **Unity**: 2022.3.15 (exact version required)
- **.NET**: .NET 8
- **IDE**: JetBrains Rider 2023.3+ (VS not officially supported)

### Setup Notes  
- Requires global VPN for Unity/NuGet package downloads
- Admin privileges needed for server HTTP services
- External Script Editor: Rider (Generate .csproj files: all unchecked)

## Common Workflows

### Adding New Features
1. Define data models in Model assembly
2. Implement logic in Hotfix assembly  
3. Add UI/presentation in HotfixView (client only)
4. Use F6/F7 for iterative development

### Server Deployment
```bash
# Linux deployment
dotnet publish -r linux-x64 --no-self-contained -c Release
# Copy Bin/ and Config/ to target server
```

### Debugging Multi-Process Issues
- Use Single Process mode for development
- Check ET/Logs/ directory for error logs
- Use REPL mode for runtime inspection

## Performance Characteristics

### Benchmarks
- 200k messages/second average throughput
- Single server supports 15k+ concurrent players (64-core/128GB tested)
- Zero-GC networking with MemoryPack serialization

### Memory Management
- Object pooling for high-frequency objects
- MemoryPack for zero-allocation serialization
- Automatic cleanup of disposed entities