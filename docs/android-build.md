# Dolphin Android 构建记录

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

## Debug/Release 共存

| | Debug | Release |
|------|------|------|
| 包名 | `org.dolphinemu.dolphinemu.debug` | `org.dolphinemu.dolphinemu` |
| 应用名 | Dolphin Debug | Dolphin Emulator |
| 图标 | 橙色渐变海豚 | 蓝色渐变海豚 |
| 图标资源 | `src/debug/res/` | `src/main/res/` |

## 注意事项

1. C++ 编译产生大量 warning（missing field initializer、deprecated API 等），均不影响构建
2. `android.enableJetifier=true` 已被 AGP 标记为 deprecated，将在 AGP 10.0 移除
3. 项目自带 `benchmark` 模块，debug 构建会包含 baseline profile 依赖
4. 签名配置使用环境变量: `KEY_STORE`、`KEY_STORE_PASSWORD`、`KEY_ALIAS`、`KEY_PASSWORD`

## 仓库空间参考

| 目录 | 大小 | 说明 |
|------|------|------|
| .git/ | 511 MB | 完整历史（48万+对象） |
| Externals/ | 749 MB | 子模块依赖源码 |
| .cxx/ (CMake 缓存) | 1,330 MB | Native 编译产物，增量构建复用 |
| Gradle build/ | 774 MB | APK + 中间产物 |

- 增量构建仅编译 Kotlin/Java 层，耗时约 11 秒
- `.cxx/` 缓存可复用，只要不 clean 或修改 CMakeLists
