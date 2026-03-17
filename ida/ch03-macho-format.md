# 第三章：Mach-O 文件格式

[返回目录](ida-pro-malware-analysis-guide.md) | [上一章](ch02-x86-64-assembly.md)

---

> Mach-O（Mach Object）是 macOS、iOS 等 Apple 平台的原生可执行文件格式，等同于 Windows 的 PE 和 Linux 的 ELF。理解 Mach-O 结构是用 IDA 分析 macOS 恶意软件的基础。

---

## 3.1 Mach-O 整体结构

一个 Mach-O 文件由三大部分组成：

```
┌─────────────────────────┐
│       Mach-O Header     │  ← 文件魔数、CPU 类型、文件类型
├─────────────────────────┤
│     Load Commands       │  ← 描述文件布局的元数据表
│  ┌───────────────────┐  │
│  │ LC_SEGMENT_64     │  │  ← 定义 __TEXT 段
│  │ LC_SEGMENT_64     │  │  ← 定义 __DATA 段
│  │ LC_SYMTAB         │  │  ← 符号表位置
│  │ LC_DYSYMTAB       │  │  ← 动态符号表
│  │ LC_LOAD_DYLIB     │  │  ← 需要加载的动态库
│  │ LC_MAIN           │  │  ← 程序入口点
│  │ ...               │  │
│  └───────────────────┘  │
├─────────────────────────┤
│       Segment Data      │  ← 实际的代码和数据
│  ┌───────────────────┐  │
│  │ __TEXT segment    │  │
│  │   __text section  │  │  ← 机器码
│  │   __stubs section │  │  ← 外部函数桩
│  │   __cstring       │  │  ← C 字符串常量
│  │   ...             │  │
│  ├───────────────────┤  │
│  │ __DATA segment    │  │
│  │   __data section  │  │  ← 已初始化全局变量
│  │   __bss section   │  │  ← 未初始化全局变量
│  │   __objc_* section│  │  ← Objective-C 运行时元数据
│  │   ...             │  │
│  ├───────────────────┤  │
│  │ __LINKEDIT segment│  │  ← 链接器元数据
│  └───────────────────┘  │
└─────────────────────────┘
```

---

## 3.2 Mach-O Header

### 查看 Header

```bash
# 使用 otool
otool -hv sample

# 输出示例
Mach header
      magic  cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC_64  X86_64        ALL  0x00     EXECUTE    17       1368   NOUNDEFS DYLDLINK TWOLEVEL PIE
```

### Header 关键字段

| 字段 | 含义 | 逆向分析关注点 |
|------|------|---------------|
| **magic** | 魔数，标识文件格式 | `0xFEEDFACF` = 64-bit Mach-O, `0xFEEDFACE` = 32-bit |
| **cputype** | CPU 架构 | `X86_64` 或 `ARM64` |
| **filetype** | 文件类型 | `MH_EXECUTE`(可执行), `MH_DYLIB`(动态库), `MH_BUNDLE`(插件) |
| **ncmds** | Load Command 数量 | 越多越复杂 |
| **flags** | 标志位 | `PIE`=地址随机化, `NOUNDEFS`=无未定义符号 |

### Universal Binary（Fat Binary）

Apple 过渡期的 Mach-O 可能是"胖二进制"，同时包含 x86-64 和 ARM64 两份代码：

```bash
# 识别 Fat Binary
file sample
# sample: Mach-O universal binary with 2 architectures:
#   x86_64: Mach-O 64-bit executable x86_64
#   arm64:  Mach-O 64-bit executable arm64

# 提取特定架构
lipo sample -thin x86_64 -output sample_x86_64
lipo sample -thin arm64 -output sample_arm64
```

> **IDA 处理 Fat Binary**：IDA 加载 Fat Binary 时会弹出对话框让你选择分析哪个架构。对于恶意软件分析，如果你的目标环境是 Intel Mac，选择 x86-64。

---

## 3.3 Load Commands

Load Commands 是 Mach-O 的元数据核心，告诉操作系统和动态链接器如何加载和执行这个文件。

### 查看 Load Commands

```bash
otool -lv sample | head -100
```

### 逆向分析最关注的 Load Commands

| Load Command | 作用 | 分析关注点 |
|-------------|------|-----------|
| `LC_SEGMENT_64` | 定义内存段 | 段的名称、虚拟地址、大小、权限 |
| `LC_MAIN` | 程序入口点偏移 | 恶意代码可能从非标准入口开始执行 |
| `LC_LOAD_DYLIB` | 需要加载的动态库 | 识别依赖了哪些系统框架/第三方库 |
| `LC_LOAD_WEAK_DYLIB` | 可选动态库 | 弱链接库不存在也不会导致加载失败 |
| `LC_RPATH` | 运行时搜索路径 | 恶意软件可能利用 @rpath 劫持 |
| `LC_CODE_SIGNATURE` | 代码签名 | 签名是否有效，签名者是谁 |
| `LC_ENCRYPTION_INFO_64` | 加密信息 | iOS 应用商店加密，macOS 少见 |

### 段（Segment）与节（Section）

每个 `LC_SEGMENT_64` 包含零个或多个 Section：

```bash
# 查看所有段和节
otool -lv sample | grep -A5 "sectname\|segname"
```

---

## 3.4 关键 Segment 和 Section 详解

### `__TEXT` 段（代码段，只读 + 可执行）

| Section | 内容 | IDA 中的呈现 |
|---------|------|-------------|
| `__text` | 编译后的机器码 | IDA View 的主要分析对象 |
| `__stubs` | 外部函数调用的跳转桩 | 短跳转指令，跳到 `__stub_helper` |
| `__stub_helper` | 懒加载桩的辅助代码 | 首次调用时触发 dyld 绑定 |
| `__cstring` | C 字符串常量 | IDA Strings 窗口中的内容来源之一 |
| `__objc_methname` | ObjC 方法名字符串 | 用于识别 ObjC 方法调用 |
| `__objc_classname` | ObjC 类名字符串 | 用于识别使用了哪些类 |
| `__const` | 常量数据 | 常量数组、只读结构体 |
| `__unwind_info` | 异常展开信息 | 帮助 IDA 识别函数边界 |

### `__DATA` 段（数据段，可读写）

| Section | 内容 | IDA 中的呈现 |
|---------|------|-------------|
| `__data` | 已初始化的全局/静态变量 | IDA 标记为 data 引用 |
| `__bss` | 未初始化的全局/静态变量 | 在文件中不占空间，加载时清零 |
| `__objc_classlist` | ObjC 类列表 | IDA 解析后可在 Structures 窗口查看 |
| `__objc_protolist` | ObjC 协议列表 | Protocol 定义 |
| `__objc_imageinfo` | ObjC 运行时版本信息 | ARC/MRC 标识 |
| `__la_symbol_ptr` | 懒加载符号指针 | 首次调用时由 dyld 填充 |
| `__nl_symbol_ptr` | 非懒加载符号指针 | 加载时立即由 dyld 填充 |
| `__got` | 全局偏移表 | 类似 ELF 的 GOT |

### `__LINKEDIT` 段

包含动态链接器需要的元数据：符号表、字符串表、重定位信息、代码签名等。不包含可执行代码，但对理解二进制的完整性很重要。

---

## 3.5 动态链接机制

### 外部函数调用流程

当恶意软件调用一个外部函数（如 `NSTask` 的 `launch` 方法），实际的调用链是：

```
代码调用 _objc_msgSend
    ↓
__stubs 中的跳转桩
    ↓  （首次调用）
__stub_helper
    ↓
dyld_stub_binder（dyld 的绑定函数）
    ↓
解析真实函数地址，写入 __la_symbol_ptr
    ↓  （后续调用直接走这里）
真实函数地址
```

在 IDA 中，你通常会看到：

```nasm
call    _objc_msgSend    ; IDA 已经帮你解析了桩的目标
```

IDA 会自动将 `__stubs` 中的跳转解析为目标函数名，所以你不需要手动追踪这个链条。但理解这个机制有助于分析那些 hook 了动态链接过程的恶意软件。

### 常见动态库依赖

```bash
# 查看依赖的动态库
otool -L sample

# 典型输出
sample:
    /usr/lib/libSystem.B.dylib              # C 标准库 + 系统调用
    /usr/lib/libobjc.A.dylib                # ObjC 运行时
    /System/Library/Frameworks/Foundation.framework/Foundation  # 基础框架
    /System/Library/Frameworks/AppKit.framework/AppKit          # GUI 框架
    /System/Library/Frameworks/Security.framework/Security      # 安全框架
    /System/Library/Frameworks/IOKit.framework/IOKit            # 硬件交互
```

> **分析线索**：如果恶意软件依赖了 `Security.framework`，可能在访问 Keychain。如果依赖了 `IOKit.framework`，可能在获取硬件信息或进行底层设备操作。

---

## 3.6 代码签名与 Entitlements

### 代码签名

macOS 使用代码签名验证二进制的来源和完整性：

```bash
# 查看签名详情
codesign -dvvv sample

# 验证签名
codesign --verify --verbose sample

# 查看签名证书链
codesign -d --extract-certificates sample
openssl x509 -inform DER -in codesign0 -text
```

签名状态对恶意软件分析的意义：

| 签名状态 | 含义 |
|---------|------|
| 有效的 Apple Developer ID | 恶意软件作者获得了开发者证书（可能被盗或购买） |
| Ad-hoc 签名 | 自签名，无 Apple 认证 |
| 无签名 | 古老样本或故意移除签名 |
| 签名无效 | 二进制被修改过 |

### Entitlements（权限声明）

```bash
codesign -d --entitlements - sample
```

恶意软件可能请求的可疑 entitlement：

- `com.apple.security.cs.allow-unsigned-executable-memory` — 允许执行未签名代码
- `com.apple.security.cs.disable-library-validation` — 允许加载未签名动态库
- `com.apple.security.automation.apple-events` — 可以控制其他应用程序

---

## 3.7 与 PE/ELF 的对照

如果你有 Windows PE 或 Linux ELF 的逆向经验，这个对照表会帮助你快速迁移：

| 概念 | Mach-O | PE (Windows) | ELF (Linux) |
|------|--------|-------------|-------------|
| 魔数 | `0xFEEDFACF` | `MZ` + `PE\0\0` | `0x7F ELF` |
| 代码段 | `__TEXT.__text` | `.text` | `.text` |
| 数据段 | `__DATA.__data` | `.data` | `.data` |
| 只读数据 | `__TEXT.__const` | `.rdata` | `.rodata` |
| 字符串常量 | `__TEXT.__cstring` | `.rdata` 中 | `.rodata` 中 |
| 导入表 | `__stubs` + Load Commands | `.idata` / IAT | `.plt` + `.got` |
| 动态链接器 | `dyld` | `ntdll.dll` | `ld-linux.so` |
| 入口点 | `LC_MAIN` | `AddressOfEntryPoint` | `e_entry` |
| 多架构 | Fat Binary | 不支持 | 不支持 |
| 包格式 | `.app` bundle | 单文件 .exe | 单文件 |

---

## 3.8 在 IDA 中查看 Mach-O 结构

IDA 加载 Mach-O 后，可以通过以下方式查看文件结构：

### 操作步骤

1. **查看 Segments**：`View > Open subviews > Segments`（快捷键 `Shift+F7`）
   - 会列出所有 Segment 及其虚拟地址范围和权限

2. **查看 Sections**：在 Segments 视图中双击某个 Segment 可以看到其下的 Sections

3. **查看 Imports**：`View > Open subviews > Imports`
   - 列出所有导入的外部函数，这是恶意软件功能分析的关键入口

4. **查看 Exports**：`View > Open subviews > Exports`
   - 对于 dylib 类型的恶意软件尤其重要

5. **查看 Strings**：`View > Open subviews > Strings`（快捷键 `Shift+F12`）
   - IDA 会从 `__cstring`、`__objc_methname` 等 section 中提取字符串

6. **查看入口点**：`Jump > Jump to entry point`（快捷键 `Ctrl+E`）

> **技巧**：加载完成后，第一件事通常是打开 Strings 窗口（`Shift+F12`），快速浏览字符串内容能给你对样本功能的第一印象。

---

## 3.9 实用命令行分析流程

在用 IDA Pro 深入分析之前，建议先用命令行工具做一次快速 triage：

```bash
#!/bin/bash
# quick_triage.sh - Mach-O 快速分类脚本
SAMPLE=$1

echo "=== 文件类型 ==="
file "$SAMPLE"

echo -e "\n=== SHA-256 ==="
shasum -a 256 "$SAMPLE"

echo -e "\n=== Mach-O Header ==="
otool -hv "$SAMPLE"

echo -e "\n=== 动态库依赖 ==="
otool -L "$SAMPLE"

echo -e "\n=== 代码签名 ==="
codesign -dvvv "$SAMPLE" 2>&1

echo -e "\n=== 可疑字符串（URL, IP, 路径）==="
strings "$SAMPLE" | grep -iE '(http[s]?://|[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+|/tmp/|/var/|/Library/|LaunchAgent|LaunchDaemon|keychain|password|encrypt|decrypt|shell|exec|curl|wget)'

echo -e "\n=== Objective-C 类名 ==="
strings "$SAMPLE" | grep -E '^[A-Z][a-zA-Z]+$' | head -30
```

---

## 本章小结

- Mach-O 由 Header、Load Commands、Segment Data 三部分组成
- `__TEXT` 段包含代码和只读数据，`__DATA` 段包含可写数据
- 外部函数通过 `__stubs` → `__stub_helper` → `dyld` 的链条完成动态绑定
- 代码签名和 Entitlements 能提供样本来源和权限意图的线索
- 分析前先用 `otool`、`file`、`strings` 做快速 triage
- IDA 加载 Mach-O 后会自动解析大部分结构，关注 Segments/Imports/Strings 视图

---

[<< 上一章：x86-64 汇编速查](ch02-x86-64-assembly.md) | [下一章：IDA Pro 界面与核心操作 >>](ch04-ida-pro-basics.md)
