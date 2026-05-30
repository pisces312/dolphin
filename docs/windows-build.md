# Dolphin Windows 桌面构建记录

## 项目信息

- **仓库**: fork 自 [dolphin-emu/dolphin](https://github.com/dolphin-emu/dolphin) → [pisces312/dolphin](https://github.com/pisces312/dolphin)
- **本地路径**: `D:\nili\3rd_party_projects\dolphin`
- **构建日期**: 2026-05-30

## 前置依赖

| 依赖 | 版本/路径 |
|------|------|
| Visual Studio Build Tools | `D:\nili\dev\visual-studio-build-tools`，MSVC 19.51 (v145 工具集) |
| CMake | 4.2.3-msvc3（VS 自带） |
| Windows SDK | 10.0.26100.0 |
| Qt 6.x | GUI 模式需要，Headless/Tests 不需要 |

## 构建方式

### 方式 1: Qt GUI 模式（完整模拟器）

```powershell
cmake -B build-win -G "Visual Studio 17 2022" -A x64 `
  -DENABLE_QT=ON `
  -DENABLE_TESTS=ON
cmake --build build-win --config RelWithDebInfo
```

产出: `Binary/x64/RelWithDebInfo/Dolphin.exe`

### 方式 2: NoGUI + Headless 模式（无 GUI）

```powershell
cmake -B build-win -G "Visual Studio 17 2022" -A x64 -T v145 `
  -DENABLE_QT=OFF `
  -DENABLE_NOGUI=ON `
  -DENABLE_HEADLESS=ON `
  -DENABLE_TESTS=ON
cmake --build build-win --config RelWithDebInfo
```

产出: `Binary/x64/RelWithDebInfo/DolphinNoGUI.exe`

> 💡 `-T v145` 指定 MSVC 工具集，因为 CMake 默认请求 v143 而环境只有 v145。

### 方式 3: 仅单元测试

```powershell
cmake -B build-test -G "Visual Studio 17 2022" -A x64 -T v145 `
  -DENABLE_QT=OFF `
  -DENABLE_TESTS=ON
cmake --build build-test --config RelWithDebInfo --target tests
```

运行测试:

```powershell
# 需先拷贝 Data/Sys/ 到测试目录
Copy-Item "Data\Sys\GameSettings" -Destination "Binary\x64\Tests\RelWithDebInfo\Sys" -Recurse -Force
Copy-Item "Data\Sys\ApprovedInis.json" -Destination "Binary\x64\Tests\RelWithDebInfo\Sys" -Force

# 运行全部测试
.\Binary\x64\Tests\RelWithDebInfo\tests.exe

# 运行单个测试
.\Binary\x64\Tests\RelWithDebInfo\tests.exe --gtest_filter="PatchAllowlist.VerifyHashes"
```

> ⚠️ 运行 tests.exe 前必须拷贝 `Data/Sys/` 到 `Binary/x64/Tests/RelWithDebInfo/Sys/`，否则 `PatchAllowlist.VerifyHashes` 会失败。

## 可用 CMake 选项

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

## 测试模式对比

| 模式 | 需要 Qt | 能加载游戏 | 有画面 | 适合 |
|------|---------|------------|--------|------|
| Qt GUI | ✅ | ✅ | ✅ | 完整体验、调试 |
| NoGUI | ❌ | ✅ | ✅ (D3D/Vulkan) | 游戏测试、无菜单栏 |
| Headless | ❌ | ✅ | ❌ | 自动化、CI、性能测试 |
| UnitTests | ❌ | ❌ | ❌ | 核心 C++ 逻辑验证 |

## 实际构建产物

| 产物 | 路径 | 大小 |
|------|------|------|
| DolphinNoGUI.exe | `Binary/x64/RelWithDebInfo/` | 17.2 MB |
| DolphinTool.exe | `Binary/x64/RelWithDebInfo/` | 14.4 MB |
| tests.exe | `Binary/x64/Tests/RelWithDebInfo/` | 15.8 MB |

## 单元测试结果

- **1345/1345 全部通过** ✅（51 test suites, ~6 秒）
- 测试覆盖: Core (CPU/JIT/IOS/DSP/Gecko/AR)、VideoCommon (VertexLoader)、Common (BitUtils/Crypto)

## 遇到的问题及解决方案

### 问题 1: MSVC 报 mGBA 编码错误

**现象**: `overrides.c` 含 UTF-8 箭头符号 ↑↓，MSVC 默认 GBK 解析出错。

**解决**: `Externals/mGBA/CMakeLists.txt` 添加 MSVC `/utf-8` 编译选项:

```cmake
if(NOT MSVC)
  target_compile_options(mgba PRIVATE -Wno-unused-parameter -Wno-unused-result -Wno-unused-variable)
else()
  target_compile_options(mgba PRIVATE /utf-8)
endif()
```

### 问题 2: CMake 要求 v143 工具集

**现象**: CMake 默认请求 v143 (VS 2022)，但环境只有 v145 (MSVC 19.51)。

**解决**: 配置时加 `-T v145` 参数。

### 问题 3: UnitTests PatchAllowlist 失败

**现象**: `PatchAllowlist.VerifyHashes` 报哈希不匹配。

**原因**: tests.exe 运行目录缺少 `Data/Sys/GameSettings` 数据文件。

**解决**: 编译后手动拷贝 `Data/Sys/` 到 `Binary/x64/Tests/RelWithDebInfo/Sys/`。

### 问题 4: MSBuild unittests target 报 ctest 失败

**现象**: 通过 `cmake --build --target unittests` 运行测试报错 exit code 8。

**原因**: MSBuild 内嵌 ctest 缺少 `-C RelWithDebInfo` 参数。

**解决**: 直接运行 `tests.exe`，不通过 MSBuild unittests target。

## 编译耗时参考

| 阶段 | 首次 | 增量 |
|------|------|------|
| CMake 配置 | ~2 分钟 | - |
| C++ 编译 (全量) | ~15 分钟 | ~1 分钟（仅改核心代码） |
| 单元测试 | - | ~6 秒 |

## 注意事项

1. glslang 15 未找到的 CMake Warning 不影响构建，项目自带 Externals/glslang 作为 fallback
2. SIMD 优化已自动启用: SSE2/SSE3/SSSE3/SSE4.1/SSE4.2/AVX/AVX2/AVX512
3. 音频后端: Cubeb + WASAPI
4. 所有外部依赖均使用 Externals/ 静态库（fmt、glslang、pugixml、enet、xxhash、zstd 等）
