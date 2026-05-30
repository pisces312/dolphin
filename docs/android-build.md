# Dolphin 构建记录（Android + Windows）

## 项目信息

- **仓库**: fork 自 [dolphin-emu/dolphin](https://github.com/dolphin-emu/dolphin) → [pisces312/dolphin](https://github.com/pisces312/dolphin)
- **本地路径**: `D:\nili\3rd_party_projects\dolphin`
- **构建日期**: 2026-05-30

## 构建配置

| 项目 | 值 |
|------|------|
| AGP | 9.1.0 |
| Gradle | 9.4.1 |
| Kotlin | 2.3.20 |
| NDK | 29.0.14206865（自动下载安装） |
| CMake | 3.22.1（SDK 自带） |
| target ABI | arm64-v8a only |
| minSdk | 24 |
| targetSdk | 35 |

## 环境准备

### 1. Fork & Clone

```bash
gh repo fork dolphin-emu/dolphin --clone=false
git clone git@github.com:pisces312/dolphin.git D:\nili\3rd_party_projects\dolphin
```

### 2. 初始化子模块（含嵌套子模块）

```bash
cd D:\nili\3rd_party_projects\dolphin
git submodule update --init --recursive
git submodule foreach --recursive "git submodule update --init --recursive"
```

> ⚠️ **重要**: 必须执行 `foreach --recursive` 确保嵌套子模块也初始化，否则 CMake 会报错。
> 已知嵌套子模块: `Externals/libadrenotools/lib/linkernsbypass`、`Externals/cpp-optparse/cpp-optparse`

### 3. Gradle 镜像配置

**gradle-wrapper.properties** — 使用腾讯镜像加速下载:

```properties
distributionUrl=https\://mirrors.cloud.tencent.com/gradle/gradle-9.4.1-bin.zip
```

**settings.gradle.kts** — 添加阿里云 Maven 镜像:

```kotlin
// pluginManagement > buildscript > repositories 前添加:
maven { url = uri("https://maven.aliyun.com/repository/central") }
maven { url = uri("https://maven.aliyun.com/repository/google") }

// pluginManagement > repositories 前添加:
maven { url = uri("https://maven.aliyun.com/repository/gradle-plugin") }
maven { url = uri("https://maven.aliyun.com/repository/google") }

// dependencyResolutionManagement > repositories 前添加:
maven { url = uri("https://maven.aliyun.com/repository/google") }
maven { url = uri("https://maven.aliyun.com/repository/central") }
```

### 4. ABI 过滤

**app/build.gradle.kts** — 仅编译 arm64-v8a:

```kotlin
abiFilters("arm64-v8a") //, "x86_64", "armeabi-v7a", "x86"
```

> 原始配置包含 x86_64，但 NDK 29 自动下载仅安装 arm64 工具链，x86_64 编译会因缺少 C 编译器而失败。

## 构建命令

```powershell
$env:GRADLE_USER_HOME = "D:/nili/dev/gradle-home"
$env:JAVA_HOME = "D:/nili/dev/AndroidStudio/jbr"
cd D:\nili\3rd_party_projects\dolphin\Source\Android
.\gradlew.bat assembleDebug
```

Release 构建:

```powershell
.\gradlew.bat assembleRelease
```

## 构建产物

| 类型 | 路径 | 大小 |
|------|------|------|
| Debug APK | `Source/Android/app/build/outputs/apk/debug/app-debug.apk` | ~20.9MB |

## 遇到的问题及解决方案

### 问题 1: 子模块文件缺失

**现象**: CMake 报错 `libadrenotools/CMakeLists.txt` 不存在。

**原因**: 浅克隆或网络中断导致子模块文件被标记为 deleted。

**解决**:

```bash
cd Externals/libadrenotools
git checkout HEAD -- .
```

### 问题 2: 嵌套子模块未初始化

**现象**: CMake 报错 `libadrenotools/lib/linkernsbypass` 目录缺少 CMakeLists.txt。

**原因**: `git submodule update --init --recursive` 只初始化一层嵌套。

**解决**:

```bash
git submodule foreach --recursive "git submodule update --init --recursive"
```

或针对特定子模块:

```bash
cd Externals/libadrenotools
git submodule update --init --recursive
```

### 问题 3: x86_64 ABI 缺少 C 编译器

**现象**: CMake 配置 x86_64 时报 `CMAKE_C_COMPILER not set`。

**原因**: NDK 29 自动下载未安装 x86_64 工具链。

**解决**: 在 `app/build.gradle.kts` 中将 abiFilters 改为仅 `arm64-v8a`。

### 问题 4: glslang 15 未找到

**现象**: CMake Warning 提示找不到 glslang 15 的配置文件。

**影响**: 无，项目自带 Externals/glslang 作为 fallback，仅影响系统库优先查找。

## 编译耗时参考

| 阶段 | 首次 | 增量 |
|------|------|------|
| C++ 原生编译 (arm64-v8a) | ~10 分钟 | 几秒（无变更时 UP-TO-DATE） |
| Kotlin/Java 编译 | ~1 分钟 | UP-TO-DATE |
| 总计 (assembleDebug) | ~13 分钟 | ~11 秒 |

## 注意事项

1. C++ 编译产生大量 warning（missing field initializer、deprecated API 等），均不影响构建
2. `android.enableJetifier=true` 已被 AGP 标记为 deprecated，将在 AGP 10.0 移除
3. 项目自带 `benchmark` 模块，debug 构建会包含 baseline profile 依赖
4. 签名配置使用环境变量: `KEY_STORE`、`KEY_STORE_PASSWORD`、`KEY_ALIAS`、`KEY_PASSWORD`

---

## 仓库空间分析

| 目录 | 大小 | 说明 |
|------|------|------|
| 仓库总计 | ~3,896 MB | |
| .git/ | 511 MB | 完整历史（48万+对象） |
| Source/ | 2,133 MB | 含 CMake 缓存 1,330 MB + Gradle 输出 774 MB |
| Externals/ | 749 MB | 子模块依赖源码 |
| CMake 缓存 (.cxx/) | 1,330 MB | Native 编译产物，增量构建复用 |
| Gradle 输出 (build/) | 774 MB | APK + 中间产物 |

### Native 构建缓存复用

- CMake 编译产物缓存在 `Source/Android/app/.cxx/` 目录
- **后续增量构建不会重新编译 C++ 代码**（只要不 clean 或修改 CMakeLists）
- 增量构建仅编译 Kotlin/Java 层，耗时约 11 秒

### Native 代码位置

| 目录 | 内容 |
|------|------|
| `Source/Core/` | C++ 模拟器核心（CPU/GPU/DSP/IOS 等） |
| `Externals/` | 第三方依赖（子模块：zlib、libpng、libusb、SFML、curl、mbedtls 等） |
| `Source/Android/app/src/main/cpp/` | Android JNI 桥接层 |
| `Source/Core/DolphinQt/` | Qt 桌面 GUI |
| `Source/Core/DolphinNoGUI/` | 无 GUI / Headless 模式 |
| `Source/Core/DolphinTool/` | CLI 工具（磁盘管理、WAD 转换等） |
| `Source/UnitTests/` | C++ 单元测试（Core/VideoCommon/Common） |

### Debug/Release 共存

| | Debug | Release |
|------|------|------|
| 包名 | `org.dolphinemu.dolphinemu.debug` | `org.dolphinemu.dolphinemu` |
| 应用名 | Dolphin Debug | Dolphin Emulator |
| 图标 | 橙色渐变海豚 | 蓝色渐变海豚 |
| 图标资源 | `src/debug/res/` | `src/main/res/` |

---

## Windows 桌面构建

Dolphin 原生支持 Windows 桌面版，可脱离 Android/手机进行 C++ 核心测试。

### 前置依赖

- Visual Studio 2022（或 Build Tools）含 C++ 桌面开发工作负载
- CMake 3.22+
- Qt 6.x（GUI 模式需要，Headless 不需要）

### 构建方式

#### 1. Qt GUI 模式（完整模拟器）

```powershell
cd D:\nili\3rd_party_projects\dolphin
cmake -B build-win -G "Visual Studio 17 2022" -A x64 `
  -DENABLE_QT=ON `
  -DENABLE_TESTS=ON
cmake --build build-win --config RelWithDebInfo
```

产出: `build-win/Binaries/Dolphin.exe` — 完整 GUI 模拟器，可直接加载游戏 ISO 测试。

#### 2. Headless 模式（无 GUI，适合自动化测试）

```powershell
cmake -B build-headless -G "Visual Studio 17 2022" -A x64 `
  -DENABLE_QT=OFF `
  -DENABLE_NOGUI=ON `
  -DENABLE_HEADLESS=ON `
  -DENABLE_TESTS=ON
cmake --build build-headless --config RelWithDebInfo
```

产出: `build-headless/Binaries/DolphinNoGUI.exe` — 无窗口运行，可命令行加载游戏。

#### 3. 仅单元测试

```powershell
cmake -B build-test -G "Visual Studio 17 2022" -A x64 `
  -DENABLE_QT=OFF `
  -DENABLE_TESTS=ON
cmake --build build-test --config RelWithDebInfo
ctest --test-dir build-test -C RelWithDebInfo
```

产出: 运行 `Source/UnitTests/` 下的 Core/VideoCommon/Common 单元测试。

### 可用 CMake 选项

| 选项 | 默认 | 说明 |
|------|------|------|
| `ENABLE_QT` | ON | Qt GUI 前端 |
| `ENABLE_NOGUI` | ON | 无 GUI 前端 |
| `ENABLE_HEADLESS` | OFF | Headless 模式（无渲染输出） |
| `ENABLE_CLI_TOOL` | ON | dolphin-tool CLI 工具 |
| `ENABLE_TESTS` | ON | 单元测试 |
| `ENABLE_VULKAN` | ON | Vulkan 渲染后端 |
| `ENABLE_LTO` | OFF | 链接时优化（编译慢、运行快） |
| `USE_MGBA` | ON | GBA 模拟联动 |
| `USE_RETRO_ACHIEVEMENTS` | ON | 成就系统 |

### 测试能力对比

| 模式 | 需要 Qt | 能加载游戏 | 有画面 | 适合 |
|------|---------|------------|--------|------|
| Qt GUI | ✅ | ✅ | ✅ | 完整体验、调试 |
| NoGUI+Headless | ❌ | ✅ | ❌ | 自动化、CI、性能测试 |
| UnitTests | ❌ | ❌ | ❌ | 核心 C++ 逻辑验证 |

> 💡 **推荐**: 日常开发用 UnitTests 快速验证核心逻辑；需要看画面时用 Qt GUI；CI/自动化用 Headless。

### 构建遇到的问题

| 问题 | 原因 | 解决 |
|------|------|------|
| MSVC 报 mGBA `overrides.c` 编码错误 | 文件含 UTF-8 箭头符号 ↑↓，MSVC 默认 GBK | `Externals/mGBA/CMakeLists.txt` 加 `/utf-8` 编译选项 |
| CMake 要求 v143 工具集 | 环境有 v145 (MSVC 19.51) | `-T v145` 指定工具集 |
| UnitTests `PatchAllowlist.VerifyHashes` 失败 | tests.exe 运行目录缺 `Data/Sys/GameSettings` | 拷贝 `Data/Sys/` 到 `Binary/x64/Tests/RelWithDebInfo/Sys/` |
| MSBuild unittests target 报 ctest 失败 | ctest 缺少 `-C RelWithDebInfo` 参数 | 直接运行 `tests.exe`，不通过 MSBuild |

### 实际构建产物

| 产物 | 路径 | 大小 |
|------|------|------|
| DolphinNoGUI.exe | `Binary/x64/RelWithDebInfo/` | 17.2 MB |
| DolphinTool.exe | `Binary/x64/RelWithDebInfo/` | 14.4 MB |
| tests.exe | `Binary/x64/Tests/RelWithDebInfo/` | 15.8 MB |
| app-debug.apk (Android) | `Source/Android/app/build/outputs/apk/debug/` | 20.9 MB |

### 单元测试结果

- **1345/1345 全部通过**（51 test suites, ~6 秒）
- 测试覆盖: Core (CPU/JIT/IOS/DSP/Gecko/AR)、VideoCommon (VertexLoader)、Common (BitUtils/Crypto)