# 第七章：IDAPython 脚本编程

[返回目录](ida-pro-malware-analysis-guide.md) | [上一章](ch06-hex-rays-decompiler.md)

---

> IDAPython 让你可以用 Python 脚本自动化 IDA 中的重复操作。在恶意软件分析中，最常见的用途是批量解密字符串、自动重命名函数、模式搜索和数据提取。

---

## 7.1 IDAPython 基础

### 运行脚本的三种方式

**方式一：Output Window 命令行**

IDA 底部的 Output Window 可以直接执行 Python 单行：

```python
Python> print(hex(here()))      # 打印当前光标地址
Python> print(idc.get_func_name(here()))  # 打印当前函数名
```

**方式二：Script File**

`File > Script file` 或快捷键 `Alt+F7`，选择一个 `.py` 文件执行。

**方式三：Script Command**

`File > Script command` 或快捷键 `Shift+F2`，打开多行脚本编辑器，适合编写和调试短脚本。

### 核心模块

IDAPython 由三个核心模块组成：

| 模块 | 用途 | 导入 |
|------|------|------|
| `idaapi` | 底层 API，提供最完整的功能 | `import idaapi` |
| `idautils` | 高层工具函数（遍历、搜索） | `import idautils` |
| `idc` | 兼容 IDC 脚本的函数接口 | `import idc` |

```python
import idaapi
import idautils
import idc
```

---

## 7.2 常用 API 速查

### 地址与导航

```python
# 获取当前光标地址
ea = idc.here()
# 等价于
ea = idaapi.get_screen_ea()

# 跳转到指定地址
idaapi.jumpto(0x100001234)

# 获取最小和最大地址
min_ea = idaapi.inf_get_min_ea()
max_ea = idaapi.inf_get_max_ea()
```

### 读取数据

```python
# 读取字节
b = idc.get_wide_byte(ea)       # 1 字节
w = idc.get_wide_word(ea)       # 2 字节
d = idc.get_wide_dword(ea)      # 4 字节
q = idc.get_qword(ea)           # 8 字节

# 读取一段字节
data = idaapi.get_bytes(ea, size)  # 返回 bytes 对象

# 读取字符串
s = idc.get_strlit_contents(ea)  # 返回 bytes 或 None
# 解码为 Python str
s = idc.get_strlit_contents(ea).decode('utf-8') if s else ""
```

### 函数操作

```python
# 获取函数名
name = idc.get_func_name(ea)

# 获取函数起始/结束地址
func = idaapi.get_func(ea)
if func:
    start = func.start_ea
    end = func.end_ea

# 遍历所有函数
for func_ea in idautils.Functions():
    name = idc.get_func_name(func_ea)
    print(f"0x{func_ea:X}: {name}")

# 重命名函数
idc.set_name(ea, "my_function_name", idc.SN_NOWARN)
```

### 交叉引用

```python
# 获取引用到 ea 的所有地址（谁引用了我）
for xref in idautils.XrefsTo(ea):
    print(f"  From: 0x{xref.frm:X}  Type: {xref.type}")

# 获取 ea 引用的所有地址（我引用了谁）
for xref in idautils.XrefsFrom(ea):
    print(f"  To: 0x{xref.to:X}  Type: {xref.type}")
```

### 注释

```python
# 设置行尾注释
idc.set_cmt(ea, "This is a comment", 0)     # 普通注释
idc.set_cmt(ea, "Repeatable comment", 1)     # 可重复注释

# 设置函数注释
idc.set_func_cmt(ea, "Function description", 0)
```

### 搜索

```python
# 搜索字节序列
pattern = "48 8D 3D ?? ?? ?? ??"  # lea rdi, [rip+???]
ea = idaapi.find_binary(min_ea, max_ea, pattern, 16, idaapi.BIN_SEARCH_FORWARD)

# 搜索文本
ea = idc.find_text(min_ea, idc.SEARCH_DOWN, 0, 0, "malware")
```

---

## 7.3 实用脚本：批量字符串解密

这是恶意软件分析中最常见的 IDAPython 应用场景。

### 场景

恶意软件使用 XOR 加密所有字符串，在运行时通过统一的解密函数解密：

```c
// 恶意软件中的调用模式
char *url = decrypt_string(encrypted_url, 35, 0x5A);
char *path = decrypt_string(encrypted_path, 22, 0x5A);
char *key = decrypt_string(encrypted_key, 16, 0x5A);
```

### 脚本

```python
"""
XOR 字符串批量解密脚本
目标：找到所有调用 decrypt_string 的位置，模拟解密，将明文注释到调用处
"""
import idaapi
import idautils
import idc

DECRYPT_FUNC_ADDR = 0x100001234  # decrypt_string 函数的地址
XOR_KEY = 0x5A

def xor_decrypt(data, key):
    """模拟 XOR 解密"""
    return bytes([b ^ key for b in data])

def process_all_calls():
    """处理所有对 decrypt_string 的调用"""
    count = 0
    for xref in idautils.XrefsTo(DECRYPT_FUNC_ADDR):
        call_addr = xref.frm

        # 向上回溯，找到传入的参数
        # 参数1 (RDI): 加密数据的地址
        # 参数2 (RSI/ESI): 数据长度
        # 参数3 (RDX/EDX): XOR key

        # 获取加密数据地址（需要向上搜索 lea rdi 指令）
        func = idaapi.get_func(call_addr)
        if not func:
            continue

        encrypted_addr = None
        data_length = None

        # 简化处理：在调用前的指令中搜索参数设置
        prev_ea = call_addr
        for _ in range(10):  # 向上搜索最多 10 条指令
            prev_ea = idc.prev_head(prev_ea)
            if prev_ea == idaapi.BADADDR:
                break

            mnem = idc.print_insn_mnem(prev_ea)
            op0 = idc.print_operand(prev_ea, 0)
            op1_val = idc.get_operand_value(prev_ea, 1)

            # 识别 lea rdi, [addr] — 加密数据地址
            if mnem == "lea" and "rdi" in op0:
                encrypted_addr = op1_val

            # 识别 mov esi, imm — 数据长度
            if mnem == "mov" and ("esi" in op0 or "rsi" in op0):
                data_length = op1_val

        if encrypted_addr and data_length and data_length < 1024:
            encrypted_data = idaapi.get_bytes(encrypted_addr, data_length)
            if encrypted_data:
                decrypted = xor_decrypt(encrypted_data, XOR_KEY)
                try:
                    plaintext = decrypted.decode('utf-8').rstrip('\x00')
                    idc.set_cmt(call_addr, f'Decrypted: "{plaintext}"', 0)
                    print(f"0x{call_addr:X}: {plaintext}")
                    count += 1
                except UnicodeDecodeError:
                    idc.set_cmt(call_addr, f'Decrypted (hex): {decrypted.hex()}', 0)

    print(f"\nDecrypted {count} strings")

process_all_calls()
```

---

## 7.4 实用脚本：批量重命名函数

### 基于字符串引用自动重命名

如果函数中引用了独特的字符串，可以据此推断函数用途：

```python
"""
基于字符串引用自动推断函数名
"""
import idautils
import idc
import idaapi

KEYWORDS = {
    "password":    "handle_password",
    "keychain":    "access_keychain",
    "LaunchAgent": "install_persistence",
    "curl":        "download_payload",
    "/bin/bash":   "execute_shell",
    "screenshot":  "capture_screen",
    "upload":      "exfiltrate_data",
    "beacon":      "c2_beacon",
}

def auto_rename_by_strings():
    renamed = 0
    for func_ea in idautils.Functions():
        func_name = idc.get_func_name(func_ea)
        # 跳过已命名的函数
        if not func_name.startswith("sub_"):
            continue

        func = idaapi.get_func(func_ea)
        if not func:
            continue

        # 收集函数内引用的所有字符串
        strings_in_func = []
        for head in idautils.Heads(func.start_ea, func.end_ea):
            for xref in idautils.XrefsFrom(head):
                s = idc.get_strlit_contents(xref.to)
                if s:
                    strings_in_func.append(s.decode('utf-8', errors='ignore'))

        # 匹配关键字
        for keyword, new_name in KEYWORDS.items():
            for s in strings_in_func:
                if keyword.lower() in s.lower():
                    # 避免重名，加上地址后缀
                    final_name = f"{new_name}_{func_ea:X}"
                    idc.set_name(func_ea, final_name, idc.SN_NOWARN)
                    print(f"Renamed {func_name} -> {final_name}")
                    renamed += 1
                    break
            else:
                continue
            break

    print(f"\nRenamed {renamed} functions")

auto_rename_by_strings()
```

---

## 7.5 实用脚本：模式搜索

### 搜索特定指令模式

```python
"""
搜索所有 'call rax' 指令（间接调用，可能是混淆或虚函数调用）
"""
import idautils
import idc

def find_indirect_calls():
    """找到所有间接调用指令"""
    results = []
    for seg_ea in idautils.Segments():
        seg = idaapi.getseg(seg_ea)
        if seg.perm & idaapi.SFL_CODE == 0:
            continue

        for head in idautils.Heads(seg.start_ea, seg.end_ea):
            if idc.print_insn_mnem(head) == "call":
                op_type = idc.get_operand_type(head, 0)
                # 操作数类型为寄存器（1）或内存引用（3,4）
                if op_type in (idc.o_reg, idc.o_mem, idc.o_displ):
                    op_str = idc.print_operand(head, 0)
                    func_name = idc.get_func_name(head)
                    results.append((head, func_name, op_str))
                    print(f"0x{head:X} in {func_name}: call {op_str}")

    print(f"\nFound {len(results)} indirect calls")
    return results

find_indirect_calls()
```

### 搜索字节序列（shellcode 特征）

```python
"""
搜索常见的 shellcode 特征字节
"""
import idaapi

patterns = {
    "syscall":       "0F 05",
    "int 0x80":      "CD 80",
    "NOP sled":      "90 90 90 90 90 90 90 90",
    "XOR decode":    "31 ?? 80 34",  # xor reg,reg; xor byte [...]
}

min_ea = idaapi.inf_get_min_ea()
max_ea = idaapi.inf_get_max_ea()

for name, pattern in patterns.items():
    ea = min_ea
    count = 0
    while ea < max_ea:
        ea = idaapi.find_binary(ea, max_ea, pattern, 16, idaapi.BIN_SEARCH_FORWARD)
        if ea == idaapi.BADADDR:
            break
        print(f"[{name}] Found at 0x{ea:X}")
        count += 1
        ea += 1

    if count:
        print(f"  Total: {count} matches\n")
```

---

## 7.6 实用脚本：数据提取

### 提取所有 C2 相关 IOC

```python
"""
从二进制中提取 IOC (Indicators of Compromise)
"""
import re
import idc
import idautils
import idaapi

def extract_iocs():
    """提取 URL、IP、邮箱、文件路径等 IOC"""
    iocs = {
        "urls": set(),
        "ips": set(),
        "emails": set(),
        "paths": set(),
        "domains": set(),
    }

    # 遍历所有字符串
    sc = idautils.Strings()
    for s in sc:
        try:
            text = str(s)
        except:
            continue

        # URL
        urls = re.findall(r'https?://[^\s<>"{}|\\^`\[\]]+', text)
        iocs["urls"].update(urls)

        # IP 地址
        ips = re.findall(r'\b(?:\d{1,3}\.){3}\d{1,3}\b', text)
        for ip in ips:
            parts = ip.split('.')
            if all(0 <= int(p) <= 255 for p in parts):
                if not ip.startswith(('0.', '127.', '255.')):
                    iocs["ips"].add(ip)

        # 文件路径
        paths = re.findall(r'(?:/(?:tmp|var|Library|Users|bin|etc|private)/\S+)', text)
        iocs["paths"].update(paths)

        # 邮箱
        emails = re.findall(r'[\w.+-]+@[\w-]+\.[\w.-]+', text)
        iocs["emails"].update(emails)

    # 输出
    print("=" * 60)
    print("IOC EXTRACTION REPORT")
    print("=" * 60)
    for category, items in iocs.items():
        if items:
            print(f"\n[{category.upper()}]")
            for item in sorted(items):
                print(f"  {item}")

    return iocs

extract_iocs()
```

---

## 7.7 Hex-Rays API（反编译器脚本）

IDAPython 也可以操作 Hex-Rays 反编译器：

```python
"""
获取函数的反编译伪代码
"""
import idaapi
import ida_hexrays

def decompile_function(ea):
    """反编译指定地址的函数，返回伪代码文本"""
    try:
        cfunc = ida_hexrays.decompile(ea)
        if cfunc:
            return str(cfunc)
    except ida_hexrays.DecompilationFailure as e:
        print(f"Decompilation failed at 0x{ea:X}: {e}")
    return None

# 使用示例
code = decompile_function(0x100001234)
if code:
    print(code)
```

### 批量导出所有函数的伪代码

```python
"""
将所有函数的反编译结果导出到文件
"""
import idautils
import idc
import ida_hexrays

output_lines = []

for func_ea in idautils.Functions():
    func_name = idc.get_func_name(func_ea)
    try:
        cfunc = ida_hexrays.decompile(func_ea)
        if cfunc:
            output_lines.append(f"// === {func_name} @ 0x{func_ea:X} ===")
            output_lines.append(str(cfunc))
            output_lines.append("")
    except:
        output_lines.append(f"// === {func_name} @ 0x{func_ea:X} === (decompilation failed)")
        output_lines.append("")

# 写入文件
output_path = "/tmp/decompiled_output.c"
with open(output_path, 'w') as f:
    f.write('\n'.join(output_lines))
print(f"Exported to {output_path}")
```

---

## 7.8 脚本开发技巧

### 1. 用 `print` 调试

IDAPython 的 `print` 输出到 IDA 的 Output Window，是最简单的调试方式。

### 2. 错误处理

```python
try:
    result = idc.get_strlit_contents(ea)
except Exception as e:
    print(f"Error at 0x{ea:X}: {e}")
```

### 3. 进度显示

对于耗时的批量操作：

```python
import idaapi

idaapi.show_wait_box("Processing... Please wait")
try:
    for i, func_ea in enumerate(idautils.Functions()):
        if idaapi.user_cancelled():
            break
        # ... 处理逻辑 ...
        if i % 100 == 0:
            idaapi.replace_wait_box(f"Processing function {i}...")
finally:
    idaapi.hide_wait_box()
```

### 4. 常用地址常量

```python
BADADDR = idaapi.BADADDR  # 0xFFFFFFFFFFFFFFFF，表示无效地址
```

### 5. IDA 版本兼容

不同 IDA 版本的 API 可能有差异。检查版本：

```python
print(f"IDA version: {idaapi.IDA_SDK_VERSION}")
```

---

## 本章小结

- IDAPython 通过 `idaapi`、`idautils`、`idc` 三个模块提供完整的 IDA 自动化能力
- 恶意软件分析中最实用的脚本场景：字符串解密、批量重命名、模式搜索、IOC 提取
- 字符串解密脚本的核心思路：找到解密函数的所有调用 → 提取参数 → 模拟解密 → 注释结果
- Hex-Rays API 允许脚本化访问反编译结果
- 养成写脚本的习惯——手动做两次以上的操作就值得写脚本

---

[<< 上一章：Hex-Rays 反编译器](ch06-hex-rays-decompiler.md) | [下一章：macOS 恶意程序常见技术 >>](ch08-macos-malware-techniques.md)
