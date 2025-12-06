# PDFium macOS 构建说明

## 构建环境要求

- macOS 12.0+
- CommandLineTools (无需完整 Xcode)
- Clang 14.0.3+
- Python 3
- 系统库: lcms2, libopenjpeg2, zlib, libjpeg

## 构建步骤

### 1. 配置构建参数

```bash
./buildtools/mac/gn gen out/Debug --args='
is_debug = true
pdf_is_standalone = true
pdf_enable_v8 = false
pdf_enable_xfa = false
is_component_build = false
use_sysroot = false
clang_use_chrome_plugins = false
use_system_zlib = true
use_system_libjpeg = true
use_system_lcms2 = true
use_system_libopenjpeg2 = true
use_custom_libcxx = false
pdf_use_partition_alloc = false
pdf_is_complete_lib = true
'
```

### 2. 构建静态库

```bash
ninja -C out/Debug pdfium
```

### 3. 生成的库文件

- `out/Debug/obj/libpdfium.a` - 完整的 PDFium 静态库 (约 132MB)

## 已修复的问题

1. **macOS SDK 版本**: 修改 `.gn` 支持 macOS SDK 13.3
2. **系统库路径**: 修复 lcms2 和 libopenjpeg2 的包含路径
3. **abseil-cpp 兼容**: 创建兼容层支持 inlined_vector 和 flat_hash_set
4. **C++ 模板错误**: 修复 typename 关键字缺失问题
5. **llvm-ar 工具**: 创建包装脚本支持 macOS ar 工具

## 依赖说明

### 手动下载的依赖

- `third_party/freetype/src` - FreeType 库
- `third_party/fast_float/src` - FastFloat 库
- `third_party/abseil-cpp/absl/container/` - Abseil 兼容头文件

### 系统库

通过 Homebrew 安装:
```bash
brew install little-cms2 openjpeg zlib libjpeg
```

## 构建产物

- 静态库: `out/Debug/obj/libpdfium.a`
- 目标文件: `out/Debug/obj/*.o` (427 个文件)

## 注意事项

- 构建需要较长时间（约 10-30 分钟，取决于机器性能）
- 确保有足够的磁盘空间（至少 2GB）
- 如果遇到网络问题，某些依赖可能需要手动下载

## 分支说明

- `main`: 官方 PDFium 主分支
- `personal-build-fixes`: 个人构建修复分支

