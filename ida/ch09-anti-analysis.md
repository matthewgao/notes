# 第九章：反分析技术与对抗

[返回目录](ida-pro-malware-analysis-guide.md) | [上一章](ch08-macos-malware-techniques.md)

---

> 高级恶意软件会主动检测和对抗逆向分析。识别这些技术是分析的必经之路。本章介绍 macOS 环境下常见的反分析技术，以及在 IDA 中如何识别和绕过它们。

---

## 9.1 反调试技术

### ptrace 反调试

最经典的 Unix/macOS 反调试手段：

```c
// 调用 ptrace 的 PT_DENY_ATTACH (31)
// 阻止调试器附加到进程
ptrace(PT_DENY_ATTACH, 0, 0, 0);
```

**IDA 中的识别**：

```nasm
; 汇编特征
mov     edi, 1Fh        ; PT_DENY_ATTACH = 31 = 0x1F
xor     esi, esi        ; pid = 0
xor     edx, edx        ; addr = 0
xor     ecx, ecx        ; data = 0
call    _ptrace
```

```c
// 反编译特征
ptrace(31, 0, 0, 0);
```

**绕过方法**：

1. **在 IDA 中 Patch**：将 `call _ptrace` 替换为 `nop` 指令
   - 在汇编视图中选中 `call` 指令
   - `Edit > Patch program > Assemble` 输入 `nop`（可能需要多个 nop 填充）

2. **通过 IDAPython 自动 Patch**：
```python
import idaapi
import idautils
import idc

# 找到所有对 ptrace 的调用并 nop 掉
for xref in idautils.XrefsTo(idc.get_name_ea_simple("_ptrace")):
    call_addr = xref.frm
    # 获取 call 指令的长度
    insn_len = idc.get_item_size(call_addr)
    # 用 NOP (0x90) 填充
    for i in range(insn_len):
        idaapi.patch_byte(call_addr + i, 0x90)
    print(f"Patched ptrace call at 0x{call_addr:X}")
```

### sysctl 调试器检测

通过 `sysctl` 查询当前进程是否正在被调试：

```c
int mib[4] = {CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()};
struct kinfo_proc info;
size_t size = sizeof(info);
sysctl(mib, 4, &info, &size, NULL, 0);

if (info.kp_proc.p_flag & P_TRACED) {
    // 检测到调试器
    exit(1);
}
```

**IDA 识别**：
- Imports 中查找 `sysctl`
- Strings 或常数中查找 `CTL_KERN`(1)、`KERN_PROC`(14)、`KERN_PROC_PID`(1)
- 查找 `P_TRACED` 标志位检测（`& 0x800`）

**绕过方法**：
- Patch 掉条件跳转——将 `jnz` (75) 改为 `jmp` (EB) 或 `nop`
- 修改比较结果——在 `test`/`and` 指令后的条件跳转处 Patch

### 时间检测（Timing Check）

通过测量代码执行时间检测单步调试：

```c
uint64_t start = mach_absolute_time();
// ... 正常代码 ...
uint64_t end = mach_absolute_time();
uint64_t elapsed = end - start;

if (elapsed > THRESHOLD) {
    // 执行时间异常长，可能在被调试
    exit(1);
}
```

**IDA 识别**：
- Imports 中查找 `mach_absolute_time`、`gettimeofday`、`clock_gettime`
- 两次时间调用之间的差值比较

**绕过方法**：Patch 时间差的比较条件，或将 threshold 值改为很大的数。

### isatty / getppid 检测

```c
// 检查是否在终端中运行（调试器通常在终端中启动）
if (isatty(STDOUT_FILENO)) {
    exit(1);
}

// 检查父进程是否是 lldb/gdb
pid_t ppid = getppid();
char ppid_name[1024];
proc_name(ppid, ppid_name, sizeof(ppid_name));
if (strstr(ppid_name, "lldb") || strstr(ppid_name, "debugserver")) {
    exit(1);
}
```

**IDA 识别**：Imports 中查找 `isatty`、`getppid`、`proc_name`。Strings 中搜索 `lldb`、`gdb`、`debugserver`。

---

## 9.2 虚拟机检测

恶意软件检测是否运行在虚拟机中，以逃避沙箱分析。

### 硬件信息检测

```c
// 通过 sysctl 获取硬件模型
char model[256];
size_t len = sizeof(model);
sysctlbyname("hw.model", model, &len, NULL, 0);
// 真实 Mac: "MacBookPro16,1"
// VMware:    "VMware..."

// 检查 CPU 品牌
sysctlbyname("machdep.cpu.brand_string", brand, &len, NULL, 0);
```

**IDA 识别**：Strings 中搜索 `hw.model`、`machdep.cpu`、`VMware`、`VirtualBox`、`QEMU`、`Parallels`。

### MAC 地址检测

虚拟化软件的网卡 MAC 地址有固定前缀：

| 虚拟化软件 | MAC 前缀 |
|-----------|---------|
| VMware | `00:0C:29`、`00:50:56` |
| VirtualBox | `08:00:27` |
| Parallels | `00:1C:42` |
| Hyper-V | `00:15:5D` |

```c
// 获取网络接口 MAC 地址并检查前缀
// 搜索 IOKit / SCNetworkInterface 相关 API
```

**IDA 识别**：Strings 中搜索这些 MAC 前缀字节。

### 进程列表检测

```c
// 检查是否有虚拟化相关进程运行
// VMware: vmware-vmx, vmtoolsd
// VirtualBox: VBoxClient, VBoxService
// 分析工具: Wireshark, tcpdump, Process Monitor
```

**IDA 识别**：Strings 中搜索虚拟化工具的进程名。

### IOKit 硬件检测

```c
// 通过 IOKit 查询硬件设备
io_service_t service = IOServiceGetMatchingService(
    kIOMasterPortDefault,
    IOServiceMatching("IOPlatformExpertDevice"));
CFStringRef serialNumber = IORegistryEntryCreateCFProperty(
    service, CFSTR("IOPlatformSerialNumber"), kCFAllocatorDefault, 0);
// 虚拟机的序列号通常有特征模式
```

**IDA 识别**：Imports 中查找 `IOServiceGetMatchingService`、`IORegistryEntryCreateCFProperty`。

### 综合 VM 检测绕过

在 IDA 中绕过 VM 检测的通用策略：

1. 找到所有 VM 检测相关函数（通过 Strings 和 Imports 搜索）
2. 用 Xref 追踪到检测逻辑的汇总点（通常是一个 `if (is_vm) exit()` 判断）
3. Patch 条件跳转，强制走"非虚拟机"路径
4. 或直接 Patch 检测函数，使其始终返回 false/0

---

## 9.3 代码混淆

### 字符串加密

最常见的混淆手段——所有字符串在二进制中以加密形式存储：

```c
// 常见加密方式
// 1. 简单 XOR（单字节 key 或循环 key）
for (int i = 0; i < len; i++)
    buf[i] ^= key;

// 2. XOR + 滚动 key
for (int i = 0; i < len; i++)
    buf[i] ^= (key + i) & 0xFF;

// 3. RC4
// 4. AES（更高级的样本）
// 5. 自定义编码表（Base64 变种）
```

**IDA 识别特征**：
- Strings 窗口中有效字符串很少
- 大量函数调用共同的"解密函数"
- 解密函数内部有循环 + XOR/查表操作

**对抗策略**：
1. 识别解密函数的算法
2. 用 IDAPython 模拟解密（参考第七章的脚本）
3. 将解密结果添加为注释

### 控制流平坦化（Control Flow Flattening）

将函数的正常控制流转换为一个大的 switch-case 循环，使控制流图变得扁平：

```c
// 原始代码
void normal() {
    step1();
    if (condition)
        step2();
    else
        step3();
    step4();
}

// 平坦化后
void flattened() {
    int state = 0;
    while (1) {
        switch (state) {
            case 0: step1(); state = condition ? 1 : 2; break;
            case 1: step2(); state = 3; break;
            case 2: step3(); state = 3; break;
            case 3: step4(); return;
        }
    }
}
```

**IDA 中的表现**：
- 函数的 CFG 有一个中心分发节点（dispatcher）
- 所有基本块都从中心节点分出
- 所有基本块都回到中心节点
- 图形看起来像一个"扁平的星型"

**对抗策略**：
1. 识别 state 变量和 dispatcher 块
2. 手动或用脚本追踪状态转换，还原原始控制流
3. 社区有 IDA 插件可以辅助反混淆（如 HexRaysDeob、D-810）

### Opaque Predicates（不透明谓词）

插入始终为真（或始终为假）的条件跳转，混淆控制流：

```nasm
; 看起来是条件跳转，实际上永远成立
mov     eax, [some_value]
imul    eax, eax           ; eax = x^2
and     eax, 1             ; x^2 的最低位
; x^2 mod 2 对于偶数 x 总是 0
jz      real_code           ; 永远跳转
; 这里的代码永远不执行（垃圾代码）
```

**IDA 中的表现**：
- 看似复杂的条件判断，但一个分支中的代码是无意义的
- 反编译器可能无法优化掉这些冗余条件

**对抗策略**：
1. 通过分析条件的数学性质判断是否为 opaque predicate
2. 将始终成立的条件跳转 Patch 为无条件跳转
3. 将始终不成立的跳转 Patch 为 nop

### 垃圾代码插入（Dead Code Insertion）

在有效代码之间插入无用指令：

```nasm
; 有效代码
mov     rdi, rbx

; 垃圾代码（不影响任何后续使用的寄存器/内存）
push    rax
xor     eax, 12345678h
add     eax, 87654321h
pop     rax

; 有效代码继续
call    _process
```

**对抗策略**：Hex-Rays 反编译器通常能自动识别和消除死代码。如果没有，关注影响后续 `call` 参数设置的指令链。

---

## 9.4 反反汇编技术

### 花指令（Junk Bytes / Anti-Disassembly）

在代码流中插入无意义的字节，干扰线性反汇编：

```nasm
    jmp     short real_code
    db      0xE8            ; 看起来像 call 指令的开头，但实际不会执行
real_code:
    mov     rdi, rbx
    call    _function
```

**IDA 中的表现**：
- 函数识别不完整（红色区域，未被识别为代码）
- 反汇编输出出现无意义的指令或错位

**对抗策略**：
1. 在 IDA 中手动修正：按 `U` 取消错误的反汇编
2. 在正确的指令起始位置按 `C` 重新反汇编
3. 按 `P` 创建函数

### 间接跳转混淆

```nasm
; 不直接跳转，而是通过寄存器计算目标地址
lea     rax, [rip + offset]
add     rax, computed_value
jmp     rax
```

**IDA 中的表现**：函数在 `jmp rax` 处被截断，IDA 无法确定跳转目标。

**对抗策略**：
1. 手动计算目标地址，在 IDA 中添加交叉引用
2. 使用 IDAPython 脚本批量解析间接跳转
3. 考虑动态分析（lldb）确认实际跳转目标

---

## 9.5 反沙箱技术

### 用户交互检测

```c
// 检查鼠标是否移动过
CGEventRef event = CGEventCreate(NULL);
CGPoint cursor = CGEventGetLocation(event);
sleep(2);
CGPoint cursor2 = CGEventGetLocation(event);
if (cursor.x == cursor2.x && cursor.y == cursor2.y) {
    // 鼠标没动 → 可能是沙箱
    exit(0);
}
```

### 系统运行时间检测

```c
// 新安装的系统（沙箱特征）运行时间很短
struct timeval boottime;
size_t len = sizeof(boottime);
int mib[2] = {CTL_KERN, KERN_BOOTTIME};
sysctl(mib, 2, &boottime, &len, NULL, 0);

time_t uptime = time(NULL) - boottime.tv_sec;
if (uptime < 600) {  // 运行不到 10 分钟
    exit(0);  // 可能是沙箱
}
```

### 文件数量检测

```c
// 真实用户的系统有大量文件，沙箱通常很"干净"
NSArray *files = [[NSFileManager defaultManager]
    contentsOfDirectoryAtPath:@"/Users" error:nil];
if ([files count] < 3) {
    exit(0);
}
```

---

## 9.6 在 IDA 中的系统化反分析对抗流程

面对使用了反分析技术的样本，建议遵循以下流程：

### Step 1: 识别反分析代码

1. 在 Strings 中搜索反分析关键字：
   `ptrace`、`sysctl`、`VMware`、`debugger`、`sandbox`
2. 在 Imports 中查找反分析 API：
   `ptrace`、`sysctl`、`mach_absolute_time`、`IOServiceGetMatchingService`
3. 在 `_main` 或 constructor 函数的开头寻找环境检查代码

### Step 2: 标记反分析函数

将识别出的反分析函数重命名：
- `sub_XXXX` → `anti_debug_ptrace`
- `sub_YYYY` → `check_vm_hardware`
- `sub_ZZZZ` → `detect_sandbox`

### Step 3: Patch 或绕过

**方法 A：Patch 条件跳转**

找到检测结果的判断处，修改跳转条件：

```nasm
; 原始代码
call    anti_debug_check
test    eax, eax
jnz     exit_if_debugger    ; 如果检测到调试器就退出

; Patch 后
call    anti_debug_check
test    eax, eax
nop                         ; 用 nop 替换 jnz
nop
nop
nop
nop
nop
```

**方法 B：Patch 检测函数返回值**

将整个检测函数替换为直接返回 0：

```nasm
; Patch 函数开头为:
xor     eax, eax    ; return 0 (未检测到)
ret
```

**方法 C：导出 Patched 二进制**

在 IDA 中完成所有 Patch 后：
1. `Edit > Patch program > Apply patches to input file`
2. 这会修改原始文件（建议先备份）
3. 在动态分析时使用 Patched 版本

### Step 4: 记录

在分析笔记中记录：
- 发现了哪些反分析技术
- 每个技术的地址和实现方式
- 如何绕过的

---

## 本章小结

- **反调试**：`ptrace(PT_DENY_ATTACH)`、`sysctl` P_TRACED 检测、时间检测是三大常见手段
- **VM 检测**：硬件模型、MAC 地址前缀、进程列表检查
- **代码混淆**：字符串加密（最常见）、控制流平坦化（最复杂）、opaque predicates
- **反反汇编**：花指令、间接跳转干扰 IDA 的静态分析
- **对抗核心方法**：Patch 条件跳转 + Patch 检测函数返回值 + IDAPython 自动化
- 系统化流程：识别 → 标记 → Patch → 记录

---

[<< 上一章：macOS 恶意程序常见技术](ch08-macos-malware-techniques.md) | [下一章：实战案例演练 >>](ch10-practical-case.md)
