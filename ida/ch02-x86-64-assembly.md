# 第二章：x86-64 汇编速查

[返回目录](ida-pro-malware-analysis-guide.md) | [上一章](ch01-environment-setup.md)

---

> 你不需要成为汇编语言专家才能做逆向分析，但你必须能**读懂**汇编。本章是一份面向逆向分析的 x86-64 汇编速查手册，重点放在"IDA 中你会反复遇到的模式"。

---

## 2.1 寄存器模型

### 通用寄存器（64-bit 模式）

x86-64 有 16 个通用寄存器，每个都可以按不同宽度访问：

```
64-bit    32-bit    16-bit    8-bit(低)   8-bit(高)
──────    ──────    ──────    ─────────   ─────────
RAX       EAX       AX        AL          AH
RBX       EBX       BX        BL          BH
RCX       ECX       CX        CL          CH
RDX       EDX       DX        DL          DH
RSI       ESI       SI        SIL         -
RDI       EDI       DI        DIL         -
RBP       EBP       BP        BPL         -
RSP       ESP       SP        SPL         -
R8        R8D       R8W       R8B         -
R9        R9D       R9W       R9B         -
R10       R10D      R10W      R10B        -
R11       R11D      R11W      R11B        -
R12       R12D      R12W      R12B        -
R13       R13D      R13W      R13B        -
R14       R14D      R14W      R14B        -
R15       R15D      R15W      R15B        -
```

### 逆向分析中的寄存器惯例

在 IDA 中读代码时，记住这些约定：

| 寄存器 | 常见用途 |
|-------|---------|
| **RAX** | 函数返回值；乘除法操作数 |
| **RCX** | 第 4 个参数（macOS/Linux）；循环计数器 |
| **RDX** | 第 3 个参数；乘除法的高位 |
| **RBX** | callee-saved，常用于存储需要跨函数调用保持的值 |
| **RSP** | 栈指针，永远指向栈顶 |
| **RBP** | 帧指针（如果未被优化掉） |
| **RSI** | 第 2 个参数 |
| **RDI** | 第 1 个参数 |
| **R8-R9** | 第 5、6 个参数 |
| **R10-R11** | caller-saved 临时寄存器 |
| **R12-R15** | callee-saved 寄存器 |
| **RIP** | 指令指针（程序计数器），不可直接赋值 |

### 标志寄存器（RFLAGS）

条件跳转的判断依据。在 IDA 中你不会直接看到 RFLAGS，但需要理解 `cmp`/`test` 指令如何设置标志：

| 标志 | 名称 | 含义 |
|-----|------|------|
| ZF | Zero Flag | 运算结果为零时置 1 |
| SF | Sign Flag | 运算结果为负时置 1 |
| CF | Carry Flag | 无符号溢出时置 1 |
| OF | Overflow Flag | 有符号溢出时置 1 |

---

## 2.2 macOS 调用约定（System V AMD64 ABI）

macOS 和 Linux 使用相同的调用约定。这是你在 IDA 中识别函数调用的关键。

### 参数传递

```
参数位置      整数/指针参数    浮点参数
─────────    ────────────    ────────
第 1 个       RDI             XMM0
第 2 个       RSI             XMM1
第 3 个       RDX             XMM2
第 4 个       RCX             XMM3
第 5 个       R8              XMM4
第 6 个       R9              XMM5
第 7+ 个      栈传递（右→左）   XMM6-XMM7, 然后栈
```

### 返回值

- 整数/指针：**RAX**（128-bit 值用 RAX:RDX）
- 浮点：**XMM0**（128-bit 浮点用 XMM0:XMM1）

### 实例：识别函数调用

当你在 IDA 中看到这样的汇编：

```nasm
lea     rdi, aHttpsMalware  ; "https://malware.example.com/payload"
xor     esi, esi            ; 0
mov     edx, 5              ; 5
call    _download_file
```

你可以推断出 C 代码大致是：

```c
download_file("https://malware.example.com/payload", 0, 5);
```

> **技巧**：IDA 的 Hex-Rays 反编译器会自动完成这种推断，但理解底层调用约定能帮你在反编译器出错时手动分析。

### Objective-C 方法调用

macOS 恶意软件大量使用 Objective-C。ObjC 方法调用通过 `objc_msgSend` 实现：

```nasm
; [NSString stringWithFormat:@"cmd: %@", command]
lea     rdi, [NSString_class]     ; self (类对象)
lea     rsi, sel_stringWithFormat ; _cmd (selector)
lea     rdx, aCmdS               ; @"cmd: %@" (第一个实际参数)
mov     rcx, [rbp+command]        ; command (第二个实际参数)
call    _objc_msgSend
; 返回值在 RAX
```

解读规则：

- `RDI` = 接收者（self，可以是类对象或实例）
- `RSI` = selector（方法名的内部表示）
- `RDX` 起 = 实际参数
- 返回值在 `RAX`

---

## 2.3 常见指令速查

### 数据传送

```nasm
mov  rax, rbx          ; rax = rbx
lea  rax, [rbx+8]      ; rax = rbx + 8 （地址计算，不访问内存）
movzx eax, byte [rsi]  ; 零扩展：读一个字节，高位补 0
movsx rax, dword [rsi] ; 符号扩展：读 4 字节，按符号位扩展到 8 字节
xchg rax, rbx          ; 交换 rax 和 rbx
push rbx               ; 将 rbx 压栈（RSP -= 8, [RSP] = RBX）
pop  rbx               ; 从栈弹出到 rbx（RBX = [RSP], RSP += 8）
```

> **LEA 的妙用**：`lea` 虽然名为"加载有效地址"，但编译器经常用它做快速算术。例如 `lea rax, [rdi+rdi*2]` 实际上是 `rax = rdi * 3`。在 IDA 中看到 `lea` 时，不一定涉及内存访问。

### 算术与逻辑

```nasm
add  rax, rbx      ; rax += rbx
sub  rax, 0x10     ; rax -= 0x10
imul rax, rbx, 5   ; rax = rbx * 5 （有符号乘法）
inc  rax           ; rax++
dec  rax           ; rax--
neg  rax           ; rax = -rax
and  rax, 0xFF     ; rax &= 0xFF （取低字节）
or   rax, rbx      ; rax |= rbx
xor  rax, rax      ; rax = 0 （清零的最快方式，你会在 IDA 中大量看到）
not  rax           ; rax = ~rax （按位取反）
shl  rax, 3        ; rax <<= 3 （等同于 rax *= 8）
shr  rax, 2        ; rax >>= 2 （逻辑右移）
sar  rax, 2        ; rax >>= 2 （算术右移，保留符号位）
rol  rax, 5        ; 循环左移 5 位（恶意软件加密常见）
ror  rax, 3        ; 循环右移 3 位
```

### 比较与测试

```nasm
cmp  rax, rbx    ; 计算 rax - rbx，设置标志位，但不保存结果
test rax, rax    ; 计算 rax & rax，设置标志位（常用于检测 rax 是否为 0）
```

### 条件跳转

`cmp` 或 `test` 之后的条件跳转，是 IDA 中控制流图的基础：

```nasm
; 无符号比较 (cmp a, b 之后)
ja   target    ; a > b    (Above)
jae  target    ; a >= b   (Above or Equal)
jb   target    ; a < b    (Below)
jbe  target    ; a <= b   (Below or Equal)

; 有符号比较 (cmp a, b 之后)
jg   target    ; a > b    (Greater)
jge  target    ; a >= b   (Greater or Equal)
jl   target    ; a < b    (Less)
jle  target    ; a <= b   (Less or Equal)

; 通用
je   target    ; a == b   (Equal / Zero)      同 jz
jne  target    ; a != b   (Not Equal / Not Zero) 同 jnz
jmp  target    ; 无条件跳转

; test rax, rax 之后
jz   target    ; rax == 0
jnz  target    ; rax != 0
js   target    ; rax < 0 （符号位为 1）
jns  target    ; rax >= 0
```

---

## 2.4 函数序言与尾声

IDA 在自动识别函数时，依赖对序言/尾声模式的识别。

### 标准序言（带帧指针）

```nasm
push    rbp            ; 保存调用者的帧指针
mov     rbp, rsp       ; 建立新的帧基准
sub     rsp, 0x30      ; 分配 48 字节的局部变量空间
; 可选：保存 callee-saved 寄存器
push    rbx
push    r12
```

### 标准尾声

```nasm
; 恢复 callee-saved 寄存器
pop     r12
pop     rbx
; 拆除栈帧
mov     rsp, rbp       ; 或 leave (等同于这两条)
pop     rbp
ret                    ; 返回到调用者
```

### 优化后的函数（无帧指针）

编译器开启优化时（`-O2`），通常省略帧指针以腾出 `RBP` 做通用寄存器：

```nasm
; 没有 push rbp / mov rbp, rsp
sub     rsp, 0x28      ; 直接分配栈空间
; ... 函数体 ...
add     rsp, 0x28      ; 直接恢复栈空间
ret
```

> 在 IDA 中，优化过的函数可能不会有清晰的帧指针。IDA 会用 `var_XX` 来标记相对于 RSP 的局部变量偏移，你需要通过 `[rsp+var_8]` 这样的标注来识别局部变量。

---

## 2.5 栈帧结构

理解栈帧布局对分析函数的局部变量至关重要：

```
高地址
┌──────────────────────────┐
│    调用者的栈帧            │
├──────────────────────────┤
│    返回地址 (call 压入)    │  ← [RBP + 8]
├──────────────────────────┤
│    旧 RBP (push rbp)     │  ← RBP 指向这里
├──────────────────────────┤
│    局部变量 1             │  ← [RBP - 8]   IDA 标记为 var_8
├──────────────────────────┤
│    局部变量 2             │  ← [RBP - 16]  IDA 标记为 var_10
├──────────────────────────┤
│    局部变量 3             │  ← [RBP - 24]  IDA 标记为 var_18
├──────────────────────────┤
│    ...                   │
├──────────────────────────┤
│    红区 (128 bytes)       │  ← RSP 指向这里
└──────────────────────────┘
低地址
```

### macOS 红区（Red Zone）

System V AMD64 ABI 定义了一个 128 字节的"红区"——RSP 以下的区域，叶子函数（不调用其他函数的函数）可以直接使用而无需调整 RSP：

```nasm
; 叶子函数可能直接使用 RSP 以下的空间
mov     [rsp-8], rax    ; 使用红区存储临时值
; 不需要 sub rsp, ... / add rsp, ...
ret
```

在 IDA 中，如果看到函数没有分配栈空间但却在 `[rsp-XX]` 处存取数据，这就是红区的使用。

---

## 2.6 常见代码模式识别

### 模式一：if-else

```c
// C 代码
if (x > 10) {
    do_something();
} else {
    do_other();
}
```

```nasm
; IDA 中的典型表现
    cmp     edi, 0Ah        ; x > 10 ?
    jle     short loc_else  ; 如果 x <= 10，跳到 else
    call    _do_something   ; then 分支
    jmp     short loc_end
loc_else:
    call    _do_other       ; else 分支
loc_end:
```

### 模式二：for 循环

```c
for (int i = 0; i < count; i++) {
    process(data[i]);
}
```

```nasm
    xor     ebx, ebx        ; i = 0
loc_loop:
    cmp     ebx, [rbp+count] ; i < count ?
    jge     short loc_end    ; 不满足则退出
    mov     rdi, [r14+rbx*8] ; data[i]
    call    _process
    inc     ebx              ; i++
    jmp     short loc_loop
loc_end:
```

### 模式三：while 循环

```nasm
loc_loop:
    test    rax, rax         ; while (ptr != NULL)
    jz      short loc_end
    ; ... 循环体 ...
    mov     rax, [rax+8]     ; ptr = ptr->next
    jmp     short loc_loop
loc_end:
```

### 模式四：switch-case（跳转表）

大型 switch 语句通常编译为跳转表：

```nasm
    cmp     eax, 5           ; switch 的 case 上界
    ja      short loc_default ; > 5 则走 default
    lea     rcx, jpt_table   ; 加载跳转表基址
    movsxd  rax, dword [rcx+rax*4] ; 读取偏移
    add     rax, rcx         ; 计算目标地址
    jmp     rax              ; 跳转到对应 case
```

> IDA 通常能自动识别跳转表并将其还原为 switch-case 结构。如果没有自动识别，可以手动在跳转指令处右键 → "Specify switch idiom"。

### 模式五：函数指针调用（虚函数/回调）

```nasm
    mov     rax, [rdi]       ; 读取对象的 vtable 指针
    call    qword [rax+0x18] ; 调用 vtable 中偏移 0x18 的虚函数
```

恶意软件经常使用函数指针来混淆控制流——你看不到直接的 `call _function_name`，而是 `call rax` 或 `call [reg+offset]`。

---

## 2.7 IDA 中的汇编阅读技巧

### 1. 先看反编译，再对照汇编

Hex-Rays 反编译器能处理大部分情况。当反编译结果不清楚时，切到汇编视图对照。

### 2. 关注 `xor reg, reg`

这几乎总是在清零寄存器，等同于 `mov reg, 0`，但更短更快。

### 3. `test rax, rax` + `jz/jnz` 是空指针检查

```nasm
test    rax, rax    ; if (ptr == NULL)
jz      error_path  ;     goto error_path
```

### 4. `rep movsb/movsq` 是内存复制

```nasm
; memcpy(rdi, rsi, rcx)
mov     rcx, count
rep     movsb           ; 逐字节复制 RCX 个字节，从 [RSI] 到 [RDI]
```

### 5. `nop` 和对齐填充

```nasm
nop                          ; 1 字节 NOP
nop dword [rax+rax+00h]     ; 多字节 NOP（对齐用，无实际含义）
```

IDA 中大量出现的 `nop` 通常只是编译器的对齐填充，可以忽略。

### 6. RIP 相对寻址

x86-64 广泛使用 RIP 相对寻址来访问全局数据：

```nasm
lea     rdi, [rip+0x1234]   ; 加载某个全局字符串的地址
; IDA 会自动解析为:
lea     rdi, aHelloWorld     ; "Hello, World!"
```

IDA 通常会自动将 RIP 相对偏移解析为符号名或字符串引用，这大大提高了可读性。

---

## 本章小结

- x86-64 有 16 个通用寄存器，macOS 使用 System V AMD64 ABI 调用约定
- 前 6 个整数参数通过 RDI, RSI, RDX, RCX, R8, R9 传递，返回值在 RAX
- ObjC 方法调用通过 `objc_msgSend`，前两个参数是 self 和 selector
- 掌握 5 种基本代码模式：if-else、for、while、switch、函数指针调用
- IDA 的自动标注（符号名、字符串引用、变量名）会帮你省去大量手动解码工作
- 遇到不确定的指令时，先看 Hex-Rays 反编译输出

---

[<< 上一章：环境搭建](ch01-environment-setup.md) | [下一章：Mach-O 文件格式 >>](ch03-macho-format.md)
