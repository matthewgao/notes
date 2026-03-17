# 附录

[返回目录](ida-pro-malware-analysis-guide.md) | [上一章](ch10-practical-case.md)

---

## 附录 A：IDA Pro 快捷键大全

### 导航

| 快捷键 | 操作 | 使用频率 |
|--------|------|---------|
| `G` | 跳转到地址 | 高 |
| `Ctrl+P` | 跳转到函数 | 高 |
| `Ctrl+E` | 跳转到入口点 | 中 |
| `X` | 查看交叉引用 (Xrefs to) | 极高 |
| `Esc` | 后退（返回上一位置） | 极高 |
| `Ctrl+Enter` | 前进 | 高 |
| `Alt+M` | 设置书签 (Mark position) | 中 |
| `Ctrl+M` | 跳转到书签 | 中 |
| `Ctrl+L` | 跳转到行号 | 低 |
| `Ctrl+S` | 跳转到段 | 低 |

### 视图切换

| 快捷键 | 操作 | 使用频率 |
|--------|------|---------|
| `空格` | 图形视图 / 文本视图切换 | 高 |
| `Tab` | IDA View / Hex View 切换 | 中 |
| `F5` | 反编译（Hex-Rays 伪代码） | 极高 |
| `Shift+F7` | Segments 窗口 | 低 |
| `Shift+F8` | Enums 窗口 | 中 |
| `Shift+F9` | Structures 窗口 | 中 |
| `Shift+F11` | Type Libraries 窗口 | 低 |
| `Shift+F12` | Strings 窗口 | 高 |

### 编辑与标注

| 快捷键 | 操作 | 使用频率 |
|--------|------|---------|
| `N` | 重命名函数/变量/标签 | 极高 |
| `;` 或 `/` | 添加注释 | 高 |
| `Ins` | 添加前置注释 | 中 |
| `Y` | 设置类型（函数签名/变量类型） | 高 |
| `D` | 切换数据类型 (byte→word→dword→qword) | 中 |
| `A` | 定义为 ASCII 字符串 | 中 |
| `C` | 定义为代码（反汇编） | 中 |
| `U` | 取消定义（还原为原始字节） | 中 |
| `P` | 在当前地址创建函数 | 中 |
| `H` | 切换十进制/十六进制显示 | 中 |
| `M` | 选择枚举成员 | 中 |
| `T` | 选择结构体偏移 | 中 |
| `*` | 定义为数组 | 低 |
| `O` | 偏移引用 | 低 |

### 搜索

| 快捷键 | 操作 | 使用频率 |
|--------|------|---------|
| `Alt+T` | 文本搜索 | 高 |
| `Ctrl+T` | 下一个文本搜索结果 | 中 |
| `Alt+B` | 字节序列搜索 | 中 |
| `Ctrl+B` | 下一个字节搜索结果 | 低 |
| `Alt+I` | 立即数搜索 | 低 |

### 其他

| 快捷键 | 操作 | 使用频率 |
|--------|------|---------|
| `Ctrl+S` | 保存数据库 | 高 |
| `Ctrl+Z` | 撤销 | 中 |
| `Ctrl+W` | 关闭当前数据库 | 低 |
| `Alt+F7` | 运行脚本文件 | 中 |
| `Shift+F2` | 打开脚本命令窗口 | 中 |
| `F2` | 设置断点（需要调试器） | 动态分析时 |

---

## 附录 B：macOS 逆向常用命令行工具速查

### 文件分析

```bash
# 文件类型识别
file sample

# Mach-O header
otool -hv sample

# Load Commands
otool -lv sample

# 动态库依赖
otool -L sample

# 符号表
nm -m sample

# 反汇编
otool -tV sample                    # 文本段反汇编
objdump -d sample                   # GNU 风格反汇编
objdump -d -M intel sample          # Intel 语法

# 字符串提取
strings sample                      # ASCII
strings -el sample                  # UTF-16 LE

# Objective-C 类信息
class-dump sample
class-dump -H -o headers/ sample    # 输出到 headers 目录
```

### 代码签名与权限

```bash
# 签名信息
codesign -dvvv sample

# 验证签名
codesign --verify --verbose=4 sample

# 查看 entitlements
codesign -d --entitlements - sample

# 移除签名
codesign --remove-signature sample

# 查看 provisioning profile
security cms -D -i embedded.mobileprovision
```

### 哈希计算

```bash
# 常用哈希
md5 sample
shasum -a 1 sample
shasum -a 256 sample

# ssdeep 模糊哈希（查找相似样本）
ssdeep sample

# 一键获取所有哈希
echo "MD5:    $(md5 -q sample)"
echo "SHA1:   $(shasum -a 1 sample | cut -d' ' -f1)"
echo "SHA256: $(shasum -a 256 sample | cut -d' ' -f1)"
```

### Fat Binary 处理

```bash
# 查看包含的架构
lipo -info sample

# 提取特定架构
lipo sample -thin x86_64 -output sample_x86
lipo sample -thin arm64 -output sample_arm

# 查看详细架构信息
lipo -detailed_info sample
```

### 动态分析辅助

```bash
# 文件系统监控
sudo fs_usage -f filesystem -w | grep sample_name

# 网络监控
sudo tcpdump -i any -w capture.pcap &

# 进程监控
ps aux | grep sample_name

# dtrace（需要 SIP 关闭）
sudo dtrace -n 'syscall:::entry /execname == "sample"/ { printf("%s", probefunc); }'

# lldb 调试
lldb sample
(lldb) breakpoint set -n main
(lldb) run
(lldb) disassemble --frame
```

### YARA 规则扫描

```bash
# 用 YARA 规则扫描文件
yara rules.yar sample

# 扫描目录
yara -r rules.yar /path/to/samples/

# 显示匹配的字符串
yara -s rules.yar sample
```

---

## 附录 C：推荐学习资源

### 书籍

| 书名 | 作者 | 说明 |
|------|------|------|
| **The Art of Mac Malware** | Patrick Wardle | macOS 恶意软件分析圣经，免费在线阅读 |
| **The IDA Pro Book** | Chris Eagle | IDA Pro 最权威的教材 |
| **Practical Malware Analysis** | Sikorski & Honig | 恶意软件分析经典入门（Windows 为主但方法论通用） |
| **Reverse Engineering for Beginners** | Dennis Yurichev | 免费，覆盖 x86/ARM 逆向基础 |
| **macOS and iOS Internals** | Jonathan Levin | macOS/iOS 系统内部机制深度解析 |

### 在线资源

| 资源 | 网址 | 说明 |
|------|------|------|
| **Objective-See Blog** | https://objective-see.org/blog.html | Patrick Wardle 的 macOS 恶意软件分析博客 |
| **The Art of Mac Malware** | https://taomm.org | 免费在线教材 |
| **Hex-Rays Blog** | https://hex-rays.com/blog/ | IDA Pro 官方博客，技巧和新功能 |
| **IDA Pro 官方文档** | https://hex-rays.com/documentation/ | 官方参考文档 |
| **Malware Traffic Analysis** | https://malware-traffic-analysis.net | 恶意软件流量分析练习 |
| **crackmes.one** | https://crackmes.one | 逆向练习题集 |

### 视频/课程

| 资源 | 说明 |
|------|------|
| **Patrick Wardle 的 Objective by the Sea 演讲** | macOS 安全年度会议，YouTube 有录像 |
| **OpenSecurityTraining2** | 免费的逆向工程和恶意软件分析课程 |
| **SANS FOR610** | 商业课程，恶意软件逆向工程（如果公司赞助的话） |

### 社区

| 社区 | 说明 |
|------|------|
| **r/ReverseEngineering** | Reddit 逆向工程社区 |
| **r/Malware** | Reddit 恶意软件分析社区 |
| **Hex-Rays 论坛** | IDA Pro 官方论坛 |
| **Objective by the Sea** | macOS 安全研究者社区 |

### IDA 插件推荐

| 插件 | 用途 | 获取方式 |
|------|------|---------|
| **HexRaysDeob** | 反控制流平坦化 | GitHub |
| **D-810** | 通用反混淆框架 | GitHub |
| **Lighthouse** | 代码覆盖率可视化 | GitHub |
| **findcrypt** | 加密常量识别 | GitHub (IDA 内置也有) |
| **LazyIDA** | 各种便捷功能增强 | GitHub |
| **FLARE IDA** | FireEye 开发的分析工具集 | GitHub |
| **Keypatch** | 汇编指令 Patch 增强 | GitHub |
| **ClassDumpIDA** | ObjC 类信息导入 | GitHub |

---

## 附录 D：常见 Mach-O 恶意行为 API 速查

### 进程/命令执行

```
system()          popen()           execve()
posix_spawn()     NSTask            fork()
NSAppleScript     OSAScript
```

### 文件操作

```
open()            read()            write()
fopen()           fread()           fwrite()
NSFileManager     NSData.writeToFile
rename()          unlink()          mkdir()
chflags()         chmod()           removexattr()
```

### 网络通信

```
NSURLSession      CFHTTPMessage     NSURLConnection
socket()          connect()         send()/recv()
getaddrinfo()     res_query()       CFSocketCreate
```

### 凭据窃取

```
SecItemCopyMatching                  SecKeychainFindGenericPassword
SecKeychainItemCopyContent           sqlite3_open (浏览器数据库)
```

### 系统信息

```
sysctlbyname()    sysctl()          getuid()/getpid()
NSProcessInfo     gethostname()     SCDynamicStoreCopyValue
IOServiceGetMatchingService
```

### 屏幕/输入捕获

```
CGWindowListCreateImage              CGEventTapCreate (键盘记录)
NSPasteboard                         AVCaptureSession (摄像头)
```

### 持久化

```
LSSharedFileListInsertItemURL        SMJobBless
writeToFile: (LaunchAgent plist)     launchctl
crontab
```

### 反分析

```
ptrace()          sysctl() + P_TRACED
mach_absolute_time()                 isatty()
getppid() + proc_name()
```

---

## 附录 E：Q&A 区

> 此区域用于记录后续学习中的问题和解答，持续更新。

---

### Q1: radare2、ssdeep、capstone、unicorn 分别能干什么？

**radare2** — 开源逆向框架，命令行版"轻量 IDA"。适合快速 triage 和脚本化批量处理：
```bash
# 列出所有函数
r2 -q -c 'aaa; afl' sample
# 反汇编 main
r2 -q -c 'aaa; pdf @ sym._main' sample
# 提取字符串并过滤
r2 -q -c 'iz' sample | grep -i "http"
# 查看谁调用了 _system
r2 -q -c 'aaa; axt @ sym.imp._system' sample
```
典型场景：50 个样本快速筛选哪些调用了 `_system`，写个 shell 循环调 radare2 几分钟搞定。

**ssdeep** — 模糊哈希工具。普通 SHA-256 改一个字节就完全不同，ssdeep 能算出两个文件的"相似度百分比"：
```bash
ssdeep -p sample_v1.bin sample_v2.bin
# 输出: sample_v2.bin matches sample_v1.bin (97) → 97% 相似，同家族变种
```
典型场景：分析完一个样本后，用 ssdeep 在样本库中批量比对找出同家族变种。

**capstone** — 反汇编引擎 Python 库，把原始字节转为汇编指令：
```python
from capstone import *
shellcode = b"\x48\x31\xc0\x48\x89\xc2\x0f\x05"
md = Cs(CS_ARCH_X86, CS_MODE_64)
for insn in md.disasm(shellcode, 0x1000):
    print(f"0x{insn.address:x}: {insn.mnemonic} {insn.op_str}")
# 0x1000: xor rax, rax
# 0x1003: mov rdx, rax
# 0x1008: syscall
```
典型场景：从内存 dump 或网络流量中提取 shellcode，用 capstone 快速反汇编查看行为。

**unicorn** — CPU 模拟器库，不真正运行恶意代码的情况下"模拟执行"机器码：
```python
from unicorn import *
from unicorn.x86_const import *
mu = Uc(UC_ARCH_X86, UC_MODE_64)
# 映射内存 → 写入解密函数机器码和密文 → 设置参数 → 模拟执行 → 读取解密结果
```
典型场景：恶意软件用复杂算法解密字符串，不想手动逆算法，直接用 unicorn 模拟执行解密函数提取明文。

**四者协作关系**：radare2 做批量快筛，ssdeep 做家族归类，capstone + unicorn 组合用于脱离 IDA 的自动化深入分析（capstone 看代码，unicorn 跑代码）。

---

*持续更新中...*

---

[<< 上一章：实战案例演练](ch10-practical-case.md) | [返回目录](ida-pro-malware-analysis-guide.md)
