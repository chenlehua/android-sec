# 车载系统 Secure Boot 安全启动深度解析

车载系统作为关键安全域（Safety-Critical Domain），其启动过程的完整性和可信性是整个系统安全的基石。Secure Boot（安全启动）通过建立从硬件信任根到上层应用的完整信任链（Chain of Trust），确保每一级启动组件的真实性和完整性，从而防止恶意代码在系统启动阶段注入。本文档从车载场景出发，全面解析 Secure Boot 的架构设计、实现机制、相关概念及行业标准。

---

## 目录

1. [Secure Boot 核心概念](#一secure-boot-核心概念)
2. [信任链架构设计](#二信任链架构设计)
3. [车载 Secure Boot 完整流程](#三车载-secure-boot-完整流程)
4. [各启动阶段详解](#四各启动阶段详解)
5. [Android Verified Boot (AVB)](#五android-verified-boot-avb)
6. [dm-verity 运行时完整性验证](#六dm-verity-运行时完整性验证)
7. [车载平台 Secure Boot 实现](#七车载平台-secure-boot-实现)
8. [Secure Boot 密钥体系](#八secure-boot-密钥体系)
9. [回滚保护机制](#九回滚保护机制)
10. [Secure Boot 与 TEE 的协同](#十secure-boot-与-tee-的协同)
11. [OTA 更新与 Secure Boot](#十一ota-更新与-secure-boot)
12. [攻击面分析与防御](#十二攻击面分析与防御)
13. [车载安全标准与合规](#十三车载安全标准与合规)
14. [概念发散与关联技术](#十四概念发散与关联技术)
15. [最佳实践与实施建议](#十五最佳实践与实施建议)

---

## 一、Secure Boot 核心概念

### 1.1 什么是 Secure Boot

Secure Boot 是一种安全机制，确保设备在启动过程中只执行经过验证的、可信的软件。其核心思想是：**每一级启动组件在将控制权交给下一级之前，必须验证下一级组件的数字签名和完整性**。

```mermaid
mindmap
  root((Secure Boot<br/>安全启动))
    信任根 Root of Trust
      硬件信任根 HW RoT
      OTP Fuse
      不可篡改
    信任链 Chain of Trust
      逐级验证
      签名校验
      哈希比对
    验证对象
      Bootloader
      Kernel
      System Image
      Vendor Image
    安全目标
      防篡改 Anti-Tampering
      防降级 Anti-Rollback
      防注入 Anti-Injection
      防克隆 Anti-Cloning
```

### 1.2 核心安全原则

```mermaid
graph LR
    subgraph "Secure Boot 三大原则"
        A["完整性<br/>Integrity"]
        B["真实性<br/>Authenticity"]
        C["不可绕过性<br/>Non-Bypassable"]
    end

    A -->|哈希校验| A1["确保代码未被篡改"]
    B -->|签名验证| B1["确保代码来源可信"]
    C -->|硬件强制| C1["确保验证流程不可跳过"]

    style A fill:#e1f5fe
    style B fill:#c8e6c9
    style C fill:#ffcdd2
```

| 原则 | 实现机制 | 说明 |
|------|---------|------|
| **完整性 (Integrity)** | SHA-256/SHA-384 哈希校验 | 确保启动镜像未被修改 |
| **真实性 (Authenticity)** | RSA/ECDSA 数字签名 | 确保镜像由合法发布者签署 |
| **不可绕过性 (Non-Bypassable)** | 硬件信任根 + Fuse 锁定 | 验证逻辑烧录在 ROM 中，不可修改 |
| **最小信任 (Minimal Trust)** | 逐级验证，最小权限 | 每级仅信任已验证的下一级 |
| **防回滚 (Anti-Rollback)** | 版本计数器 (Fuse/RPMB) | 防止降级到已知漏洞版本 |

### 1.3 Secure Boot vs Verified Boot vs Trusted Boot

```mermaid
graph TB
    subgraph "启动安全机制对比"
        subgraph "Secure Boot 安全启动"
            SB1["验证失败 → 停止启动"]
            SB2["强制策略 Enforcement"]
            SB3["BootROM → Bootloader"]
        end

        subgraph "Verified Boot 验证启动"
            VB1["验证失败 → 警告/限制"]
            VB2["可报告策略 Reporting"]
            VB3["Bootloader → OS"]
        end

        subgraph "Trusted Boot 可信启动"
            TB1["记录度量值 → PCR"]
            TB2["远程证明 Attestation"]
            TB3["基于 TPM/TEE"]
        end
    end

    SB3 -->|"信任传递"| VB3
    VB3 -->|"度量记录"| TB3

    style SB1 fill:#ffcdd2
    style VB1 fill:#fff3e0
    style TB1 fill:#c8e6c9
```

| 特性 | Secure Boot | Verified Boot | Trusted Boot |
|------|------------|---------------|--------------|
| **验证时机** | 硬件层启动 | OS 加载阶段 | 全程度量 |
| **失败行为** | 停止启动 | 警告/降级 | 记录度量值 |
| **验证方** | BootROM (硬件) | Bootloader | TPM/TEE |
| **典型实现** | ARM Secure Boot | Android AVB | TCG Trusted Boot |
| **车载应用** | SoC 启动 | Android 镜像验证 | HSM 远程证明 |

---

## 二、信任链架构设计

### 2.1 完整信任链架构

信任链（Chain of Trust）是 Secure Boot 的核心架构思想。从不可篡改的硬件信任根（Hardware Root of Trust）出发，每一级组件验证下一级，形成完整的信任传递链路。

```mermaid
graph TB
    subgraph "Hardware Root of Trust 硬件信任根"
        ROM["BootROM<br/>芯片内置代码<br/>不可修改"]
        FUSE["OTP Fuse<br/>一次性可编程熔丝<br/>存储根公钥哈希"]
        ROM --- FUSE
    end

    subgraph "Primary Bootloader 一级引导"
        PBL["PBL / Pre-Loader<br/>芯片厂商固件<br/>初始化基础硬件"]
    end

    subgraph "Secondary Bootloader 二级引导"
        SBL["SBL / LK / ABL<br/>平台引导程序<br/>初始化 DRAM/eMMC"]
    end

    subgraph "Trusted OS 可信操作系统"
        TEE_OS["TEE OS<br/>OP-TEE / Trusty / QSEE / Kinibi<br/>安全世界操作系统"]
    end

    subgraph "Android OS 操作系统"
        KERNEL["Linux Kernel<br/>内核镜像"]
        VENDOR["Vendor Image<br/>厂商定制"]
        SYSTEM["System Image<br/>Android 框架"]
        INIT["Init / SELinux<br/>系统初始化"]
    end

    subgraph "Applications 应用层"
        APP["系统应用 & 第三方应用"]
    end

    ROM -->|"① 验证 PBL 签名"| PBL
    PBL -->|"② 验证 SBL 签名"| SBL
    SBL -->|"③ 验证 TEE 签名"| TEE_OS
    SBL -->|"④ 验证 Kernel 签名"| KERNEL
    KERNEL -->|"⑤ dm-verity 验证"| SYSTEM
    KERNEL -->|"⑥ dm-verity 验证"| VENDOR
    SYSTEM --> INIT
    INIT -->|"⑦ APK 签名验证"| APP

    style ROM fill:#ff8a80
    style FUSE fill:#ff8a80
    style PBL fill:#ffcdd2
    style SBL fill:#ffcdd2
    style TEE_OS fill:#ce93d8
    style KERNEL fill:#c8e6c9
    style VENDOR fill:#c8e6c9
    style SYSTEM fill:#c8e6c9
    style INIT fill:#e1f5fe
    style APP fill:#e1f5fe
```

### 2.2 信任传递模型

```mermaid
sequenceDiagram
    participant ROM as BootROM<br/>(HW RoT)
    participant PBL as Primary<br/>Bootloader
    participant SBL as Secondary<br/>Bootloader
    participant TEE as TEE OS
    participant Kernel as Linux Kernel
    participant AVB as AVB / dm-verity
    participant System as Android System

    Note over ROM: 芯片上电复位 (Power-On Reset)
    ROM->>ROM: 从 OTP Fuse 读取根公钥哈希
    ROM->>PBL: 验证签名 + 哈希比对
    activate PBL
    Note over PBL: ✓ 验证通过，执行 PBL

    PBL->>PBL: 初始化时钟、PMIC、DDR
    PBL->>SBL: 验证签名 + 加载到 DDR
    activate SBL
    Note over SBL: ✓ 验证通过，执行 SBL

    SBL->>SBL: 初始化 eMMC/UFS、Display
    SBL->>TEE: 验证 TEE 镜像签名并加载
    activate TEE
    Note over TEE: 安全世界初始化完成

    SBL->>Kernel: 验证 boot.img 签名
    activate Kernel
    Note over Kernel: ✓ 验证通过，内核启动

    Kernel->>AVB: 加载 AVB 元数据
    activate AVB
    AVB->>AVB: 校验 vbmeta 签名
    AVB->>System: 配置 dm-verity 映射表
    activate System
    Note over System: ✓ 运行时持续验证

    SBL-->>TEE: 传递 Root of Trust 状态
    TEE-->>TEE: 记录 Verified Boot State
    Note over TEE: 用于 Key Attestation
```

### 2.3 信任根 (Root of Trust) 详解

硬件信任根是整个信任链的起点，其安全性直接决定了整个系统的安全性。

```mermaid
graph TB
    subgraph "Root of Trust 组成"
        subgraph "不可变组件 Immutable"
            BROM["BootROM Code<br/>芯片出厂固化<br/>128KB - 256KB"]
            FUSE["OTP Fuse<br/>一次性编程<br/>根公钥 SHA-256"]
            HW_ENGINE["硬件加密引擎<br/>AES/SHA/RSA/ECC<br/>硬件加速器"]
        end

        subgraph "安全存储 Secure Storage"
            RPMB["RPMB 分区<br/>Replay Protected<br/>Memory Block"]
            EFUSE["eFuse 区域<br/>设备唯一标识<br/>安全配置位"]
        end

        subgraph "安全状态 Security State"
            SEC_BOOT["Secure Boot Enable<br/>安全启动使能位"]
            JTAG["JTAG Disable<br/>调试接口关闭"]
            DAA["DAA Disable<br/>下载代理认证关闭"]
            LOCK["Device Lock<br/>设备锁定状态"]
        end
    end

    BROM --> SEC_BOOT
    FUSE --> BROM
    HW_ENGINE --> BROM
    RPMB --> EFUSE
    SEC_BOOT --- JTAG
    JTAG --- DAA
    DAA --- LOCK

    style BROM fill:#ff8a80
    style FUSE fill:#ff8a80
    style HW_ENGINE fill:#ff8a80
    style RPMB fill:#ffcdd2
    style EFUSE fill:#ffcdd2
    style SEC_BOOT fill:#fff3e0
    style JTAG fill:#fff3e0
    style DAA fill:#fff3e0
    style LOCK fill:#fff3e0
```

**OTP Fuse 关键字段：**

| Fuse 字段 | 位宽 | 说明 |
|-----------|------|------|
| `ROOT_KEY_HASH` | 256 bit | 根公钥 SHA-256 哈希值 |
| `SECURE_BOOT_EN` | 1 bit | 安全启动使能标志 |
| `JTAG_DISABLE` | 1 bit | JTAG 调试接口禁用 |
| `DAA_DISABLE` | 1 bit | Download Agent Authentication 禁用 |
| `SBC_EN` | 1 bit | Secure Boot Certificate 使能 |
| `ANTI_ROLLBACK_VER` | 32 bit | 防回滚版本计数器 |
| `DEVICE_ID` | 128 bit | 设备唯一标识 |

---

## 三、车载 Secure Boot 完整流程

### 3.1 上电启动完整时序

```mermaid
graph TB
    subgraph "Phase 0: 硬件复位"
        P0_1["Power-On Reset"]
        P0_2["CPU 进入 BootROM 执行"]
        P0_3["安全状态初始化"]
        P0_1 --> P0_2 --> P0_3
    end

    subgraph "Phase 1: BootROM 执行"
        P1_1["读取 OTP Fuse 安全配置"]
        P1_2["初始化安全引擎 (Crypto Engine)"]
        P1_3["从 Boot Device 加载 PBL"]
        P1_4["验证 PBL 签名"]
        P1_5{"验证通过？"}
        P1_6["跳转到 PBL"]
        P1_7["终止启动 / 进入恢复模式"]
        P1_1 --> P1_2 --> P1_3 --> P1_4 --> P1_5
        P1_5 -->|Yes| P1_6
        P1_5 -->|No| P1_7
    end

    subgraph "Phase 2: PBL / Pre-Loader"
        P2_1["初始化 PMIC / DDR / Clock"]
        P2_2["加载 SBL / LK 到 DDR"]
        P2_3["验证 SBL 签名"]
        P2_4{"验证通过？"}
        P2_5["跳转到 SBL"]
        P2_6["终止启动"]
        P2_1 --> P2_2 --> P2_3 --> P2_4
        P2_4 -->|Yes| P2_5
        P2_4 -->|No| P2_6
    end

    subgraph "Phase 3: SBL / Bootloader"
        P3_1["初始化 eMMC/UFS / Display"]
        P3_2["加载并验证 TEE OS"]
        P3_3["加载并验证 boot.img"]
        P3_4["读取 vbmeta / 验证 AVB"]
        P3_5["检查回滚保护版本"]
        P3_6["传递 Verified Boot State 到 TEE"]
        P3_7["跳转到 Kernel"]
        P3_1 --> P3_2 --> P3_3 --> P3_4 --> P3_5 --> P3_6 --> P3_7
    end

    subgraph "Phase 4: Kernel & Android"
        P4_1["Linux Kernel 启动"]
        P4_2["挂载 dm-verity 保护的分区"]
        P4_3["Init 进程启动"]
        P4_4["SELinux 策略加载"]
        P4_5["Android Framework 启动"]
        P4_1 --> P4_2 --> P4_3 --> P4_4 --> P4_5
    end

    P0_3 --> P1_1
    P1_6 --> P2_1
    P2_5 --> P3_1
    P3_7 --> P4_1

    style P0_1 fill:#ff8a80
    style P1_5 fill:#fff9c4
    style P2_4 fill:#fff9c4
    style P1_7 fill:#ef5350,color:#fff
    style P2_6 fill:#ef5350,color:#fff
    style P4_5 fill:#c8e6c9
```

### 3.2 签名验证核心算法流程

```mermaid
flowchart TD
    START["开始验证"]
    LOAD["加载待验证镜像"]
    LOAD_CERT["加载镜像签名证书"]

    HASH["计算镜像 SHA-256 哈希"]
    EXTRACT_SIG["提取数字签名"]
    EXTRACT_PK["提取签名公钥"]

    VERIFY_CHAIN["验证证书链<br/>Certificate Chain"]
    CHECK_ROOT{"公钥哈希 =<br/>OTP Fuse 根哈希？"}

    VERIFY_SIG["使用公钥验证签名"]
    SIG_VALID{"签名有效？"}

    HASH_CMP["比对计算哈希与签名哈希"]
    HASH_MATCH{"哈希匹配？"}

    CHECK_ROLLBACK["检查防回滚版本"]
    ROLLBACK_OK{"版本 ≥ 最小版本？"}

    SUCCESS["✓ 验证成功<br/>允许执行"]
    FAIL["✗ 验证失败<br/>终止启动"]

    START --> LOAD --> LOAD_CERT
    LOAD_CERT --> EXTRACT_SIG
    LOAD_CERT --> EXTRACT_PK
    LOAD --> HASH

    EXTRACT_PK --> VERIFY_CHAIN --> CHECK_ROOT
    CHECK_ROOT -->|Yes| VERIFY_SIG
    CHECK_ROOT -->|No| FAIL

    EXTRACT_SIG --> VERIFY_SIG
    VERIFY_SIG --> SIG_VALID
    SIG_VALID -->|Yes| HASH_CMP
    SIG_VALID -->|No| FAIL

    HASH --> HASH_CMP
    HASH_CMP --> HASH_MATCH
    HASH_MATCH -->|Yes| CHECK_ROLLBACK
    HASH_MATCH -->|No| FAIL

    CHECK_ROLLBACK --> ROLLBACK_OK
    ROLLBACK_OK -->|Yes| SUCCESS
    ROLLBACK_OK -->|No| FAIL

    style SUCCESS fill:#c8e6c9
    style FAIL fill:#ef5350,color:#fff
    style CHECK_ROOT fill:#fff9c4
    style SIG_VALID fill:#fff9c4
    style HASH_MATCH fill:#fff9c4
    style ROLLBACK_OK fill:#fff9c4
```

---

## 四、各启动阶段详解

### 4.1 BootROM 阶段

BootROM 是芯片出厂时固化在 SoC 内部 ROM 中的代码，是整个信任链中唯一不可修改的起点。

```mermaid
graph TB
    subgraph "BootROM 内部结构"
        subgraph "固化代码区 (Read-Only)"
            CODE1["启动向量 Reset Vector"]
            CODE2["时钟初始化 Clock Init"]
            CODE3["安全引擎初始化 Crypto Init"]
            CODE4["Boot Device 检测<br/>(eMMC/UFS/SPI NOR)"]
            CODE5["镜像加载器 Image Loader"]
            CODE6["签名验证器 Signature Verifier"]
        end

        subgraph "OTP Fuse 接口"
            FUSE_R["Fuse Read Controller"]
            ROOT_HASH["Root Public Key Hash"]
            SEC_CFG["Security Configuration"]
        end

        subgraph "硬件加密引擎"
            SHA["SHA-256 Engine"]
            RSA_E["RSA Engine (2048/4096)"]
            ECC_E["ECC Engine (P-256)"]
        end
    end

    CODE1 --> CODE2 --> CODE3 --> CODE4 --> CODE5 --> CODE6
    CODE6 --> FUSE_R
    FUSE_R --> ROOT_HASH
    FUSE_R --> SEC_CFG
    CODE6 --> SHA
    CODE6 --> RSA_E

    style CODE1 fill:#ff8a80
    style CODE6 fill:#ff8a80
    style ROOT_HASH fill:#ffcdd2
    style SHA fill:#ce93d8
    style RSA_E fill:#ce93d8
```

**BootROM 关键特性：**

- **不可修改**：代码在芯片制造时写入 Mask ROM，无法通过任何软件手段修改
- **最小化**：代码量极小（通常 64KB-256KB），减少攻击面
- **确定性执行**：无外部输入干扰，每次执行路径完全一致
- **硬件加速**：内置的密码学引擎提供高效签名验证能力

### 4.2 Bootloader 阶段

Bootloader 通常分为两级：PBL（Primary Bootloader）和 SBL（Secondary Bootloader）/ LK（Little Kernel）。

```mermaid
stateDiagram-v2
    [*] --> PBL_Start: BootROM 验证通过

    state "PBL (Primary Bootloader)" as PBL {
        PBL_Start --> HW_Init: 硬件初始化
        HW_Init --> DDR_Init: DDR 训练 & 初始化
        DDR_Init --> Load_SBL: 加载 SBL 到 DDR
        Load_SBL --> Verify_SBL: 验证 SBL 签名
        Verify_SBL --> Jump_SBL: 跳转到 SBL
    }

    state "SBL / LK (Secondary Bootloader)" as SBL {
        Jump_SBL --> Platform_Init: 平台初始化
        Platform_Init --> Display_Init: 显示初始化 (开机画面)
        Display_Init --> Storage_Init: 存储初始化
        Storage_Init --> Load_TEE: 加载 & 验证 TEE OS
        Load_TEE --> Load_Boot: 加载 & 验证 boot.img
        Load_Boot --> AVB_Verify: AVB 验证流程
        AVB_Verify --> Pass_VBState: 传递 Verified Boot State
        Pass_VBState --> Jump_Kernel: 跳转到 Kernel
    }

    state "决策分支" as Decision {
        Jump_Kernel --> Normal_Boot: 正常启动
        Jump_Kernel --> Recovery: 恢复模式
        Jump_Kernel --> Fastboot: Fastboot 模式
    }

    Normal_Boot --> [*]
    Recovery --> [*]
    Fastboot --> [*]
```

**SBL/LK 关键职责：**

```mermaid
graph LR
    subgraph "Bootloader 职责矩阵"
        subgraph "硬件初始化"
            H1["DDR 控制器"]
            H2["eMMC/UFS 控制器"]
            H3["Display 控制器"]
            H4["USB 控制器"]
            H5["PMIC 电源管理"]
        end

        subgraph "安全功能"
            S1["Secure Boot 验证"]
            S2["AVB 元数据校验"]
            S3["回滚保护检查"]
            S4["TEE 镜像加载"]
            S5["Verified Boot State 传递"]
        end

        subgraph "启动管理"
            B1["Boot Mode 选择"]
            B2["A/B Slot 管理"]
            B3["Fastboot 协议"]
            B4["Recovery 模式"]
            B5["内核命令行构建"]
        end
    end

    style S1 fill:#ffcdd2
    style S2 fill:#ffcdd2
    style S3 fill:#ffcdd2
    style S4 fill:#ffcdd2
    style S5 fill:#ffcdd2
```

### 4.3 Kernel 加载与验证

boot.img 包含 Linux Kernel、DTB（Device Tree Blob）和可选的 ramdisk，作为一个整体进行签名验证。

```mermaid
graph TB
    subgraph "boot.img 结构"
        HEADER["Boot Image Header<br/>magic: ANDROID!<br/>kernel_size, ramdisk_size<br/>page_size, header_version"]
        KERNEL_IMG["Kernel Image<br/>(zImage / Image.gz)"]
        RAMDISK["Ramdisk<br/>(init / fstab)"]
        DTB["Device Tree Blob<br/>(硬件描述)"]
        VB_SIG["VBMeta Signature<br/>(AVB 签名)"]
    end

    HEADER --> KERNEL_IMG
    KERNEL_IMG --> RAMDISK
    RAMDISK --> DTB
    DTB --> VB_SIG

    subgraph "验证流程"
        V1["Bootloader 读取 vbmeta"]
        V2["解析 vbmeta 描述符"]
        V3["计算各组件哈希"]
        V4["验证签名"]
        V5["设置 dm-verity 参数"]
    end

    VB_SIG --> V1
    V1 --> V2 --> V3 --> V4 --> V5

    style HEADER fill:#e1f5fe
    style VB_SIG fill:#ffcdd2
    style V4 fill:#c8e6c9
```

---

## 五、Android Verified Boot (AVB)

### 5.1 AVB 2.0 架构总览

Android Verified Boot 2.0（也称 AVB）是 Google 在 Android 8.0 引入的统一验证启动框架，是车载 Android 系统使用的标准验证机制。

```mermaid
graph TB
    subgraph "AVB 2.0 架构"
        subgraph "VBMeta 结构"
            VBMETA["vbmeta.img<br/>核心元数据"]
            VBMETA_SYS["vbmeta_system.img"]
            VBMETA_VND["vbmeta_vendor.img"]
        end

        subgraph "Hash Descriptor 哈希描述符"
            HD_BOOT["boot 分区哈希"]
            HD_DTBO["dtbo 分区哈希"]
            HD_INIT["init_boot 分区哈希"]
        end

        subgraph "Hashtree Descriptor 哈希树描述符"
            HT_SYS["system 分区哈希树"]
            HT_VND["vendor 分区哈希树"]
            HT_PROD["product 分区哈希树"]
        end

        subgraph "Chained Partition 链式分区"
            CP_SYS["→ vbmeta_system"]
            CP_VND["→ vbmeta_vendor"]
        end
    end

    VBMETA --> HD_BOOT
    VBMETA --> HD_DTBO
    VBMETA --> HD_INIT
    VBMETA --> CP_SYS
    VBMETA --> CP_VND
    CP_SYS --> VBMETA_SYS
    CP_VND --> VBMETA_VND
    VBMETA_SYS --> HT_SYS
    VBMETA_SYS --> HT_PROD
    VBMETA_VND --> HT_VND

    style VBMETA fill:#ffcdd2
    style VBMETA_SYS fill:#fff3e0
    style VBMETA_VND fill:#fff3e0
```

### 5.2 VBMeta 数据结构

```mermaid
graph TB
    subgraph "VBMeta Image 内部结构"
        subgraph "Header (256 bytes)"
            H1["Magic: 'AVB0'"]
            H2["Header Version: 1.2"]
            H3["Auth Block Size"]
            H4["Aux Block Size"]
            H5["Algorithm: SHA256_RSA4096"]
            H6["Key Size / Signature Size"]
            H7["Rollback Index"]
            H8["Flags"]
        end

        subgraph "Auth Block 认证块"
            AB1["Hash Digest<br/>(SHA-256 of Header+Aux)"]
            AB2["Signature<br/>(RSA-4096 签名)"]
        end

        subgraph "Aux Block 辅助块"
            AX1["Public Key (DER)"]
            AX2["Public Key Metadata"]
            AX3["Descriptors[]<br/>描述符数组"]
        end

        subgraph "Descriptor Types 描述符类型"
            D1["HashDescriptor<br/>小分区直接哈希"]
            D2["HashtreeDescriptor<br/>大分区 Merkle Tree"]
            D3["PropertyDescriptor<br/>属性键值对"]
            D4["ChainPartitionDescriptor<br/>链式分区委托"]
            D5["KernelCmdlineDescriptor<br/>内核命令行"]
        end
    end

    H3 --> AB1
    H4 --> AX1
    AX3 --> D1
    AX3 --> D2
    AX3 --> D3
    AX3 --> D4
    AX3 --> D5
    AB1 --> AB2

    style H1 fill:#e1f5fe
    style AB2 fill:#ffcdd2
    style AX1 fill:#c8e6c9
```

### 5.3 AVB 验证流程

```mermaid
sequenceDiagram
    participant BL as Bootloader
    participant Storage as eMMC/UFS
    participant Crypto as Crypto Engine
    participant TEE as TEE

    BL->>Storage: 读取 vbmeta 分区
    Storage-->>BL: vbmeta.img

    BL->>BL: 解析 VBMeta Header
    BL->>BL: 提取 Algorithm / Public Key

    BL->>Crypto: 验证 vbmeta 签名
    Note over Crypto: RSA-4096 + SHA-256
    Crypto-->>BL: 签名有效

    BL->>BL: 检查 Rollback Index
    Note over BL: 比对 RPMB 中的最小版本

    loop 遍历 Hash Descriptors
        BL->>Storage: 读取分区 (boot/dtbo)
        Storage-->>BL: 分区数据
        BL->>Crypto: 计算 SHA-256
        Crypto-->>BL: 哈希值
        BL->>BL: 比对描述符中的哈希
    end

    loop 遍历 Chained Partitions
        BL->>Storage: 读取链式 vbmeta
        Storage-->>BL: vbmeta_xxx.img
        BL->>BL: 递归验证签名和描述符
    end

    BL->>TEE: 传递 Verified Boot State
    Note over TEE: VERIFIED / SELF_SIGNED /<br/>UNVERIFIED / FAILED

    BL->>BL: 构造 kernel cmdline
    Note over BL: androidboot.verifiedbootstate=green<br/>androidboot.vbmeta.device_state=locked
```

### 5.4 Verified Boot State（启动状态）

```mermaid
stateDiagram-v2
    [*] --> Green: 设备锁定 + 验证通过
    [*] --> Yellow: 设备锁定 + 自签名密钥
    [*] --> Orange: 设备解锁
    [*] --> Red: 验证失败

    state "Green (绿色)" as Green {
        GD: 设备锁定 (Locked)
        GV: 使用 OEM 嵌入密钥验证通过
        GT: 信任级别：完全可信
    }

    state "Yellow (黄色)" as Yellow {
        YD: 设备锁定 (Locked)
        YV: 使用自签名密钥验证通过
        YT: 信任级别：用户自签
        YW: 启动时显示警告指纹
    }

    state "Orange (橙色)" as Orange {
        OD: 设备解锁 (Unlocked)
        OV: 可能跳过验证
        OT: 信任级别：不可信
        OW: 启动时显示解锁警告
    }

    state "Red (红色)" as Red {
        RD: 验证失败
        RV: 镜像签名无效
        RT: 信任级别：拒绝
        RW: 拒绝启动或强制恢复
    }

    Green --> [*]: 正常运行
    Yellow --> [*]: 警告后运行
    Orange --> [*]: 警告后运行
    Red --> [*]: 拒绝启动
```

**车载场景下的启动状态策略：**

| 状态 | 颜色 | 车载策略 | TEE 行为 |
|------|------|---------|---------|
| **VERIFIED** | Green | 正常启动，全功能 | 正常提供密钥服务 |
| **SELF_SIGNED** | Yellow | 工厂/开发模式，受限功能 | 限制高安全等级密钥 |
| **UNLOCKED** | Orange | 开发模式，显示警告 | 禁止访问生产密钥 |
| **FAILED** | Red | **禁止启动**，车载场景不允许 | 拒绝所有服务 |

---

## 六、dm-verity 运行时完整性验证

### 6.1 dm-verity 工作原理

dm-verity 是 Linux Device-Mapper 的一个目标（target），用于对块设备进行透明的完整性验证。它使用 Merkle Hash Tree（默克尔哈希树）实现高效的按需验证。

```mermaid
graph TB
    subgraph "dm-verity Merkle Tree 结构"
        ROOT["Root Hash<br/>根哈希 (32 bytes)<br/>存储在 vbmeta 中"]

        L2_1["Hash Node L2-0"]
        L2_2["Hash Node L2-1"]

        L1_1["Hash Node L1-0"]
        L1_2["Hash Node L1-1"]
        L1_3["Hash Node L1-2"]
        L1_4["Hash Node L1-3"]

        B1["Data Block 0<br/>4KB"]
        B2["Data Block 1<br/>4KB"]
        B3["Data Block 2<br/>4KB"]
        B4["Data Block 3<br/>4KB"]
        B5["Data Block 4<br/>4KB"]
        B6["Data Block 5<br/>4KB"]
        B7["Data Block 6<br/>4KB"]
        B8["Data Block 7<br/>4KB"]
    end

    ROOT --> L2_1
    ROOT --> L2_2
    L2_1 --> L1_1
    L2_1 --> L1_2
    L2_2 --> L1_3
    L2_2 --> L1_4
    L1_1 --> B1
    L1_1 --> B2
    L1_2 --> B3
    L1_2 --> B4
    L1_3 --> B5
    L1_3 --> B6
    L1_4 --> B7
    L1_4 --> B8

    style ROOT fill:#ffcdd2
    style L2_1 fill:#fff3e0
    style L2_2 fill:#fff3e0
    style L1_1 fill:#e1f5fe
    style L1_2 fill:#e1f5fe
    style L1_3 fill:#e1f5fe
    style L1_4 fill:#e1f5fe
    style B1 fill:#c8e6c9
    style B2 fill:#c8e6c9
    style B3 fill:#c8e6c9
    style B4 fill:#c8e6c9
    style B5 fill:#c8e6c9
    style B6 fill:#c8e6c9
    style B7 fill:#c8e6c9
    style B8 fill:#c8e6c9
```

### 6.2 dm-verity 读取验证流程

```mermaid
sequenceDiagram
    participant App as 应用/进程
    participant VFS as VFS 层
    participant DM as dm-verity
    participant Block as 块设备
    participant Cache as Hash Cache

    App->>VFS: read("/system/app/foo.apk")
    VFS->>DM: 请求读取 Block N

    DM->>Cache: 查找 Block N 的哈希缓存
    alt 缓存命中
        Cache-->>DM: 已验证标记
        DM->>Block: 直接读取 Block N
        Block-->>DM: Block N 数据
        DM-->>VFS: 返回数据
    else 缓存未命中
        DM->>Block: 读取 Block N 数据
        Block-->>DM: Block N 数据
        DM->>DM: 计算 SHA-256(Block N)

        DM->>Block: 读取哈希树节点
        Block-->>DM: L1 Hash Node
        DM->>DM: 验证计算哈希 vs 树节点哈希

        loop 逐级向上验证
            DM->>Block: 读取上级哈希节点
            Block-->>DM: Lx Hash Node
            DM->>DM: 验证当前节点哈希
        end

        DM->>DM: 最终比对 Root Hash

        alt 验证通过
            DM->>Cache: 缓存验证结果
            DM-->>VFS: 返回数据
        else 验证失败
            DM-->>VFS: 返回 I/O Error (-EIO)
            Note over DM: 触发 dm-verity 错误策略
        end
    end

    VFS-->>App: 数据 / 错误
```

### 6.3 dm-verity 错误处理策略

```mermaid
graph TB
    subgraph "dm-verity 错误策略"
        ERROR["检测到完整性错误"]

        subgraph "eio 模式 (默认)"
            EIO1["返回 I/O Error"]
            EIO2["对应文件不可读"]
            EIO3["系统继续运行"]
        end

        subgraph "restart 模式"
            RST1["重启设备"]
            RST2["重新验证"]
            RST3["可能进入恢复"]
        end

        subgraph "panic 模式"
            PNC1["内核 panic"]
            PNC2["系统立即停止"]
            PNC3["车载安全最高级"]
        end

        subgraph "logging 模式"
            LOG1["仅记录日志"]
            LOG2["不阻断操作"]
            LOG3["开发调试用"]
        end
    end

    ERROR --> EIO1
    ERROR --> RST1
    ERROR --> PNC1
    ERROR --> LOG1

    style ERROR fill:#ef5350,color:#fff
    style PNC1 fill:#ffcdd2
    style RST1 fill:#fff3e0
    style EIO1 fill:#e1f5fe
    style LOG1 fill:#c8e6c9
```

**车载场景策略选择：**

| 错误策略 | 适用场景 | 安全等级 | 车载建议 |
|---------|---------|---------|---------|
| `panic` | 仪表盘、ADAS 域 | 最高 | 功能安全关键系统 |
| `restart` | IVI 娱乐域 | 高 | 非实时安全系统 |
| `eio` | 普通应用分区 | 中 | 可容忍部分不可用 |
| `logging` | 开发/测试阶段 | 低 | 仅限开发环境 |

---

## 七、车载平台 Secure Boot 实现

### 7.1 MTK（联发科）平台

```mermaid
graph TB
    subgraph "MTK 平台 Secure Boot 架构"
        subgraph "BootROM Layer"
            MTK_ROM["MTK BootROM<br/>芯片内置"]
            MTK_FUSE["OTP eFuse<br/>ROOT_KEY_HASH<br/>SECURE_BOOT_EN"]
        end

        subgraph "Pre-Loader Layer"
            MTK_PL["Pre-Loader<br/>(pl.img)<br/>DDR 初始化"]
            MTK_DA["Download Agent<br/>(DA 授权)"]
        end

        subgraph "LK Layer (Little Kernel)"
            MTK_LK["LK Bootloader<br/>(lk.img)<br/>平台初始化"]
            MTK_LOGO["Logo 显示"]
            MTK_AB["A/B Slot 管理"]
        end

        subgraph "TEE Layer"
            MTK_TBASE["Trustonic Kinibi<br/>(tee.img)<br/>TEE OS"]
            MTK_TA["Trusted Applications<br/>Keymaster / Gatekeeper"]
        end

        subgraph "Android Layer"
            MTK_KERNEL["Linux Kernel<br/>(boot.img)"]
            MTK_VB["Android Verified Boot<br/>(vbmeta.img)"]
        end
    end

    MTK_ROM -->|"1. RSA验证"| MTK_PL
    MTK_FUSE -.-> MTK_ROM
    MTK_PL -->|"2. RSA验证"| MTK_LK
    MTK_PL -->|"DA认证"| MTK_DA
    MTK_LK -->|"3. RSA验证"| MTK_TBASE
    MTK_LK -->|"4. AVB验证"| MTK_KERNEL
    MTK_LK --> MTK_LOGO
    MTK_LK --> MTK_AB
    MTK_LK --> MTK_VB
    MTK_TBASE --> MTK_TA

    style MTK_ROM fill:#ff8a80
    style MTK_FUSE fill:#ff8a80
    style MTK_PL fill:#ffcdd2
    style MTK_LK fill:#ffcdd2
    style MTK_TBASE fill:#ce93d8
    style MTK_KERNEL fill:#c8e6c9
```

**MTK 平台 Secure Boot 配置要点：**

- **eFuse 烧写**：通过 SP Flash Tool 或产线工具烧写 `ROOT_KEY_HASH` 和 `SECURE_BOOT_EN`
- **签名工具链**：使用 MTK Sign Tool（`SignTool.exe`）对各级镜像进行签名
- **证书格式**：MTK 使用自定义证书格式（不同于标准 X.509）
- **DA 认证**：Download Agent 需要授权认证，防止非法刷机
- **Rollback 保护**：通过 eFuse 中的 Anti-Rollback Version 实现

### 7.2 高通（Qualcomm）平台

```mermaid
graph TB
    subgraph "Qualcomm 平台 Secure Boot 架构"
        subgraph "PBL Layer"
            QC_PBL["Qualcomm PBL<br/>Primary Boot Loader<br/>SoC 内置 ROM"]
            QC_FUSE["QFPROM eFuse<br/>OEM_PK_HASH<br/>SECURE_BOOT_EN"]
        end

        subgraph "XBL Layer (Extensible Boot Loader)"
            QC_XBL["XBL Core<br/>(xbl.elf)<br/>DDR/Clock/PMIC"]
            QC_XBL_CFG["XBL Config<br/>(xbl_config.elf)"]
            QC_XBL_SEC["XBL SEC<br/>Security Extensions"]
        end

        subgraph "ABL Layer (Android Boot Loader)"
            QC_ABL["ABL<br/>(abl.elf)<br/>基于 UEFI EDK2"]
            QC_AVB["AVB Library"]
            QC_DISPLAY["Display / Splash"]
        end

        subgraph "QTEE Layer"
            QC_TEE["QTEE / TZ<br/>(tz.mbn)<br/>Secure World OS"]
            QC_DEVCFG["Device Config<br/>(devcfg.mbn)"]
            QC_KEYMASTER["Keymaster TA<br/>(keymaster64.mbn)"]
        end

        subgraph "HLOS Layer"
            QC_KERNEL["Linux Kernel<br/>(boot.img)"]
            QC_VENDOR["Vendor Boot<br/>(vendor_boot.img)"]
        end
    end

    QC_PBL -->|"1. 验证 XBL"| QC_XBL
    QC_FUSE -.-> QC_PBL
    QC_XBL -->|"2. 验证 QTEE"| QC_TEE
    QC_XBL -->|"3. 验证 ABL"| QC_ABL
    QC_XBL --> QC_XBL_CFG
    QC_XBL --> QC_XBL_SEC
    QC_TEE --> QC_DEVCFG
    QC_TEE --> QC_KEYMASTER
    QC_ABL -->|"4. AVB 验证"| QC_KERNEL
    QC_ABL --> QC_AVB
    QC_ABL --> QC_DISPLAY
    QC_ABL --> QC_VENDOR

    style QC_PBL fill:#ff8a80
    style QC_FUSE fill:#ff8a80
    style QC_XBL fill:#ffcdd2
    style QC_ABL fill:#ffcdd2
    style QC_TEE fill:#ce93d8
    style QC_KERNEL fill:#c8e6c9
    style QC_VENDOR fill:#c8e6c9
```

**高通平台 Secure Boot 特性：**

- **QFPROM**：Qualcomm Fuse 编程区域，存储 OEM 公钥哈希和安全配置
- **XBL 架构**：基于 UEFI 的可扩展 Bootloader，模块化设计
- **PIK/OAK 签名体系**：PIK (Platform Integration Key) + OAK (OEM Application Key) 双密钥体系
- **Sectools 工具链**：`sectools` 套件提供签名、加密、证书管理等功能
- **RPMB 防回滚**：通过 eMMC/UFS 的 RPMB 分区存储回滚计数器

### 7.3 MTK vs 高通平台对比

```mermaid
graph TB
    subgraph "平台 Secure Boot 对比"
        subgraph "MTK 平台"
            M1["BootROM → Pre-Loader → LK → Kernel"]
            M2["签名: MTK Sign Tool"]
            M3["Fuse: MTK eFuse"]
            M4["TEE: Trustonic Kinibi"]
            M5["Bootloader: Little Kernel"]
        end

        subgraph "Qualcomm 平台"
            Q1["PBL → XBL → ABL → Kernel"]
            Q2["签名: Sectools"]
            Q3["Fuse: QFPROM"]
            Q4["TEE: QTEE (TrustZone)"]
            Q5["Bootloader: UEFI EDK2"]
        end
    end

    style M1 fill:#e1f5fe
    style Q1 fill:#fff3e0
```

| 对比项 | MTK 平台 | Qualcomm 平台 |
|-------|---------|--------------|
| **Boot Chain** | BootROM → PL → LK → Kernel | PBL → XBL → ABL → Kernel |
| **Bootloader 架构** | Little Kernel (LK) | UEFI EDK2 |
| **TEE OS** | Trustonic Kinibi | QTEE (TrustZone) |
| **Fuse 技术** | MTK eFuse | QFPROM |
| **签名工具** | MTK Sign Tool | Sectools |
| **签名算法** | RSA-2048/SHA-256 | RSA-2048/4096, ECDSA P-384 |
| **证书格式** | MTK 自定义格式 | X.509 证书链 |
| **回滚保护** | eFuse Counter | RPMB + QFPROM |
| **调试保护** | SLA (Serial Link Auth) | Secure Debug Policy |
| **OTA 支持** | A/B with LK | A/B with ABL |

---

## 八、Secure Boot 密钥体系

### 8.1 密钥层级结构

```mermaid
graph TB
    subgraph "Secure Boot 密钥层级"
        subgraph "Level 0: 根密钥 Root Key"
            ROOT_KEY["OEM Root Key<br/>(RSA-4096 / ECC P-384)<br/>最高安全等级<br/>HSM 离线保管"]
            ROOT_HASH["Root Key Hash<br/>(SHA-256)<br/>烧写到 OTP Fuse"]
        end

        subgraph "Level 1: 签名密钥 Signing Keys"
            PLAT_KEY["Platform Key<br/>平台签名密钥<br/>签署 Bootloader"]
            TEE_KEY["TEE Signing Key<br/>TEE 签名密钥<br/>签署 TEE OS"]
            AVB_KEY["AVB Key<br/>AVB 签名密钥<br/>签署 vbmeta"]
        end

        subgraph "Level 2: 分区密钥 Partition Keys"
            BOOT_KEY["Boot Key<br/>签署 boot.img"]
            VENDOR_KEY["Vendor Key<br/>签署 vendor.img"]
            SYSTEM_KEY["System Key<br/>签署 system.img"]
            OTA_KEY["OTA Key<br/>签署 OTA 包"]
        end

        subgraph "Level 3: 运行时密钥"
            APK_KEY["APK 签名密钥"]
            TLS_KEY["TLS 证书密钥"]
            DRM_KEY["DRM 密钥"]
        end
    end

    ROOT_KEY --> ROOT_HASH
    ROOT_KEY --> PLAT_KEY
    ROOT_KEY --> TEE_KEY
    ROOT_KEY --> AVB_KEY
    PLAT_KEY --> BOOT_KEY
    AVB_KEY --> VENDOR_KEY
    AVB_KEY --> SYSTEM_KEY
    AVB_KEY --> OTA_KEY
    SYSTEM_KEY -.-> APK_KEY
    TEE_KEY -.-> TLS_KEY
    TEE_KEY -.-> DRM_KEY

    style ROOT_KEY fill:#ff8a80
    style ROOT_HASH fill:#ff8a80
    style PLAT_KEY fill:#ffcdd2
    style TEE_KEY fill:#ffcdd2
    style AVB_KEY fill:#ffcdd2
    style BOOT_KEY fill:#fff3e0
    style VENDOR_KEY fill:#fff3e0
    style SYSTEM_KEY fill:#fff3e0
    style OTA_KEY fill:#fff3e0
```

### 8.2 密钥生命周期管理

```mermaid
stateDiagram-v2
    [*] --> Generated: 密钥生成
    Generated --> Stored: 安全存储
    Stored --> Distributed: 分发部署
    Distributed --> InUse: 生产签名
    InUse --> Rotated: 密钥轮换
    Rotated --> Stored: 新密钥存储
    InUse --> Revoked: 密钥撤销
    Revoked --> [*]: 生命周期终止

    state "密钥生成 Generated" as Generated {
        G1: HSM 内部生成
        G2: 真随机数源 (TRNG)
        G3: 密钥仪式 (Key Ceremony)
    }

    state "安全存储 Stored" as Stored {
        S1: 离线 HSM (根密钥)
        S2: 在线 HSM (签名密钥)
        S3: 安全备份 (M-of-N 分片)
    }

    state "生产签名 InUse" as InUse {
        U1: CI/CD 签名流水线
        U2: 审计日志记录
        U3: 访问控制 (RBAC)
    }

    state "密钥轮换 Rotated" as Rotated {
        R1: 定期轮换 (年度)
        R2: 事件触发轮换
        R3: 向后兼容过渡
    }

    state "密钥撤销 Revoked" as Revoked {
        V1: CRL 发布
        V2: 远程吊销通知
        V3: 设备端更新
    }
```

### 8.3 密钥安全存储方案

```mermaid
graph TB
    subgraph "密钥存储安全等级"
        subgraph "Level 1: 离线 HSM"
            HSM_OFFLINE["离线 HSM 保险柜<br/>存储根密钥<br/>M-of-N 分片"]
            HSM_OFFLINE_DESC["• 物理隔离, 无网络连接<br/>• 多人授权访问 (Quorum)<br/>• 防篡改审计日志"]
        end

        subgraph "Level 2: 在线签名服务"
            HSM_ONLINE["在线 HSM / KMS<br/>存储签名密钥<br/>自动化签名"]
            HSM_ONLINE_DESC["• CI/CD 集成<br/>• API 访问控制<br/>• 速率限制"]
        end

        subgraph "Level 3: 设备端"
            DEVICE_FUSE["OTP Fuse<br/>根公钥哈希<br/>不可修改"]
            DEVICE_RPMB["RPMB<br/>回滚计数器<br/>重放保护"]
        end
    end

    HSM_OFFLINE -->|"离线签名"| HSM_ONLINE
    HSM_ONLINE -->|"签名镜像"| DEVICE_FUSE
    HSM_ONLINE -.->|"更新计数器"| DEVICE_RPMB

    style HSM_OFFLINE fill:#ff8a80
    style HSM_ONLINE fill:#ffcdd2
    style DEVICE_FUSE fill:#fff3e0
    style DEVICE_RPMB fill:#fff3e0
```

---

## 九、回滚保护机制

### 9.1 回滚攻击与防御

回滚攻击（Rollback Attack）是指攻击者将设备固件降级到包含已知漏洞的旧版本，从而利用已修复的漏洞进行攻击。

```mermaid
graph TB
    subgraph "回滚攻击场景"
        ATTACKER["攻击者"]
        OLD_FW["旧版固件<br/>v1.0 (含漏洞)"]
        NEW_FW["新版固件<br/>v2.0 (已修复)"]
        DEVICE["目标设备"]
        EXPLOIT["利用已知漏洞<br/>获取系统权限"]

        ATTACKER -->|"1. 获取旧版合法固件"| OLD_FW
        ATTACKER -->|"2. 刷入旧版固件"| DEVICE
        DEVICE -->|"3. 运行有漏洞版本"| EXPLOIT
    end

    subgraph "防回滚机制"
        VER_COUNTER["版本计数器<br/>Anti-Rollback Counter"]
        FUSE_VER["eFuse 版本号<br/>单向递增<br/>不可回退"]
        RPMB_VER["RPMB 版本号<br/>安全存储<br/>重放保护"]

        CHECK["固件版本 ≥ 计数器版本？"]
        ALLOW["✓ 允许启动"]
        DENY["✗ 拒绝启动"]

        VER_COUNTER --> FUSE_VER
        VER_COUNTER --> RPMB_VER
        CHECK --> ALLOW
        CHECK --> DENY
    end

    DEVICE -.->|"防御"| VER_COUNTER

    style EXPLOIT fill:#ef5350,color:#fff
    style DENY fill:#ef5350,color:#fff
    style ALLOW fill:#c8e6c9
    style FUSE_VER fill:#ffcdd2
    style RPMB_VER fill:#ffcdd2
```

### 9.2 回滚保护实现方式

```mermaid
graph LR
    subgraph "eFuse 方式"
        EF1["优点: 不可篡改"]
        EF2["缺点: 次数有限<br/>(通常 32-64 次)"]
        EF3["适用: 关键组件<br/>Bootloader / TEE"]
    end

    subgraph "RPMB 方式"
        RP1["优点: 次数不受限"]
        RP2["缺点: 依赖 eMMC/UFS<br/>安全性略低于 eFuse"]
        RP3["适用: 系统分区<br/>OTA 版本管理"]
    end

    subgraph "混合方式 (推荐)"
        HY1["eFuse: Bootloader 版本"]
        HY2["RPMB: System/Vendor 版本"]
        HY3["TEE: 统一管理接口"]
    end

    style EF1 fill:#c8e6c9
    style RP1 fill:#c8e6c9
    style HY1 fill:#e1f5fe
    style HY2 fill:#e1f5fe
    style HY3 fill:#e1f5fe
```

### 9.3 AVB Rollback Index 机制

```mermaid
sequenceDiagram
    participant BL as Bootloader
    participant VBMETA as vbmeta
    participant RPMB as RPMB 存储
    participant TEE as TEE

    BL->>VBMETA: 读取 rollback_index
    Note over VBMETA: rollback_index = 5

    BL->>TEE: 请求读取 RPMB
    TEE->>RPMB: 安全读取存储的最小版本
    RPMB-->>TEE: stored_index = 4
    TEE-->>BL: min_rollback_index = 4

    BL->>BL: 比较: 5 ≥ 4
    Note over BL: ✓ 版本合法，允许启动

    alt 更新成功后
        BL->>TEE: 更新 RPMB: index = 5
        TEE->>RPMB: 安全写入新版本号
        Note over RPMB: stored_index = 5<br/>旧版本将无法启动
    end

    Note over BL,RPMB: 下次启动时，v4 及以下版本<br/>将被拒绝
```

---

## 十、Secure Boot 与 TEE 的协同

### 10.1 协同架构

Secure Boot 和 TEE 在车载系统中紧密协同：Secure Boot 确保启动时的完整性，TEE 确保运行时的安全性。两者共同构建了从启动到运行的全生命周期安全保障。

```mermaid
graph TB
    subgraph "启动时安全 (Secure Boot)"
        SB_CHAIN["信任链验证<br/>BootROM → Kernel"]
        SB_STATE["Verified Boot State<br/>Green/Yellow/Orange/Red"]
        SB_ROLLBACK["回滚保护<br/>版本计数器"]
    end

    subgraph "过渡阶段 (Handoff)"
        HANDOFF["Root of Trust 传递<br/>VB State → TEE"]
    end

    subgraph "运行时安全 (TEE)"
        TEE_KM["Keymaster / KeyMint<br/>密钥管理"]
        TEE_GK["Gatekeeper<br/>密码验证"]
        TEE_FP["Fingerprint TA<br/>指纹识别"]
        TEE_ATT["Key Attestation<br/>密钥证明"]
        TEE_DRM["DRM (Widevine)<br/>内容保护"]
    end

    subgraph "安全决策"
        DEC_1["VB State = Green?<br/>→ 允许密钥操作"]
        DEC_2["VB State = Orange?<br/>→ 禁止生产密钥"]
        DEC_3["Rollback Detected?<br/>→ 销毁密钥"]
    end

    SB_CHAIN --> SB_STATE
    SB_STATE --> HANDOFF
    SB_ROLLBACK --> HANDOFF
    HANDOFF --> TEE_KM
    HANDOFF --> TEE_ATT

    TEE_KM --> DEC_1
    TEE_KM --> DEC_2
    TEE_KM --> DEC_3

    TEE_ATT -.->|"报告 VB State"| SB_STATE

    style SB_CHAIN fill:#ffcdd2
    style HANDOFF fill:#fff3e0
    style TEE_KM fill:#ce93d8
    style TEE_ATT fill:#ce93d8
```

### 10.2 Key Attestation 中的 Secure Boot 信息

TEE 中的 Key Attestation 机制会将 Secure Boot 状态嵌入到证书中，使远程服务器能够验证设备的启动安全状态。

```mermaid
graph TB
    subgraph "Key Attestation 证书中的 Root of Trust"
        subgraph "RootOfTrust Extension (OID: 1.3.6.1.4.1.11129.2.1.17)"
            ROT_VK["verifiedBootKey<br/>Verified Boot 公钥<br/>(OCTET_STRING)"]
            ROT_LOCKED["deviceLocked<br/>设备锁定状态<br/>(BOOLEAN)"]
            ROT_STATE["verifiedBootState<br/>启动验证状态<br/>(ENUM: 0-3)"]
            ROT_HASH["verifiedBootHash<br/>启动状态哈希<br/>(OCTET_STRING)"]
        end

        subgraph "远程验证"
            SERVER["远程服务器"]
            CHECK_CHAIN["验证证书链<br/>→ Google Root CA"]
            CHECK_ROT["检查 RootOfTrust<br/>确认设备安全状态"]
            DECISION["安全决策<br/>是否信任设备"]
        end
    end

    ROT_VK --> CHECK_ROT
    ROT_LOCKED --> CHECK_ROT
    ROT_STATE --> CHECK_ROT
    ROT_HASH --> CHECK_ROT
    CHECK_CHAIN --> CHECK_ROT
    CHECK_ROT --> DECISION
    DECISION --> SERVER

    style ROT_VK fill:#e1f5fe
    style ROT_STATE fill:#ffcdd2
    style SERVER fill:#c8e6c9
```

### 10.3 Secure Boot 状态对 TEE 服务的影响

```mermaid
graph LR
    subgraph "VB State 影响矩阵"
        subgraph "Green (已验证)"
            G_KM["✓ Keymaster 完整功能"]
            G_ATT["✓ Key Attestation"]
            G_DRM["✓ Widevine L1"]
            G_PAY["✓ 支付功能"]
        end

        subgraph "Yellow (自签名)"
            Y_KM["✓ Keymaster 受限"]
            Y_ATT["△ Attestation 标记自签"]
            Y_DRM["✗ Widevine 降级 L3"]
            Y_PAY["✗ 支付功能禁用"]
        end

        subgraph "Orange (已解锁)"
            O_KM["△ Keymaster 基础功能"]
            O_ATT["△ Attestation 标记解锁"]
            O_DRM["✗ Widevine 禁用"]
            O_PAY["✗ 支付功能禁用"]
        end
    end

    style G_KM fill:#c8e6c9
    style G_ATT fill:#c8e6c9
    style G_DRM fill:#c8e6c9
    style G_PAY fill:#c8e6c9
    style Y_KM fill:#fff9c4
    style Y_ATT fill:#fff9c4
    style Y_DRM fill:#ffcdd2
    style Y_PAY fill:#ffcdd2
    style O_KM fill:#fff9c4
    style O_ATT fill:#fff9c4
    style O_DRM fill:#ffcdd2
    style O_PAY fill:#ffcdd2
```

---

## 十一、OTA 更新与 Secure Boot

### 11.1 OTA 安全更新流程

车载系统的 OTA（Over-The-Air）更新必须与 Secure Boot 机制紧密配合，确保更新包的合法性和更新后系统的可信性。

```mermaid
sequenceDiagram
    participant Cloud as OTA 云端
    participant Device as 车载设备
    participant BL as Bootloader
    participant AB as A/B 分区管理
    participant RPMB as RPMB

    Cloud->>Cloud: 构建更新包
    Cloud->>Cloud: 签名 (OTA Key)
    Cloud->>Cloud: 生成 payload + vbmeta

    Cloud->>Device: 推送 OTA 更新通知

    Device->>Cloud: 下载更新包
    Device->>Device: 验证 OTA 包签名
    Note over Device: 使用 OTA 公钥验证

    alt 签名有效
        Device->>AB: 确定目标 Slot (inactive)
        Note over AB: 当前: Slot A (active)<br/>目标: Slot B (inactive)

        Device->>AB: 写入新镜像到 Slot B
        Device->>AB: 写入新 vbmeta 到 Slot B
        Device->>AB: 标记 Slot B 为 bootable

        Device->>Device: 触发重启

        BL->>AB: 读取 Slot 状态
        BL->>BL: 选择 Slot B 启动
        BL->>BL: 执行 Secure Boot 验证
        Note over BL: 完整信任链验证

        alt 验证通过
            BL->>RPMB: 更新回滚计数器
            BL->>AB: 标记 Slot B 为 successful
            Note over Device: ✓ 更新成功，正常运行
        else 验证失败
            BL->>AB: 回退到 Slot A
            Note over Device: ✗ 更新回退，使用旧版本
        end
    else 签名无效
        Device->>Device: 丢弃更新包
        Note over Device: ✗ 更新包不合法
    end
```

### 11.2 A/B 分区与 Secure Boot

```mermaid
graph TB
    subgraph "A/B 分区布局"
        subgraph "Slot A (Active)"
            A_BOOT["boot_a"]
            A_SYSTEM["system_a"]
            A_VENDOR["vendor_a"]
            A_VBMETA["vbmeta_a"]
            A_DTBO["dtbo_a"]
        end

        subgraph "Slot B (Inactive)"
            B_BOOT["boot_b"]
            B_SYSTEM["system_b"]
            B_VENDOR["vendor_b"]
            B_VBMETA["vbmeta_b"]
            B_DTBO["dtbo_b"]
        end

        subgraph "共享分区 (Non-A/B)"
            USERDATA["userdata"]
            MISC["misc<br/>(Boot Control)"]
            PERSIST["persist"]
            METADATA["metadata"]
        end

        subgraph "安全存储"
            RPMB_S["RPMB<br/>Rollback Counters"]
            EFUSE_S["eFuse<br/>Root Key Hash"]
        end
    end

    A_VBMETA -->|"验证"| A_BOOT
    A_VBMETA -->|"验证"| A_SYSTEM
    A_VBMETA -->|"验证"| A_VENDOR

    B_VBMETA -->|"验证"| B_BOOT
    B_VBMETA -->|"验证"| B_SYSTEM
    B_VBMETA -->|"验证"| B_VENDOR

    MISC -->|"Slot 选择"| A_VBMETA
    MISC -->|"Slot 选择"| B_VBMETA

    RPMB_S -->|"版本检查"| A_VBMETA
    RPMB_S -->|"版本检查"| B_VBMETA

    style A_BOOT fill:#c8e6c9
    style A_SYSTEM fill:#c8e6c9
    style A_VENDOR fill:#c8e6c9
    style A_VBMETA fill:#c8e6c9
    style B_BOOT fill:#e1f5fe
    style B_SYSTEM fill:#e1f5fe
    style B_VENDOR fill:#e1f5fe
    style B_VBMETA fill:#e1f5fe
    style RPMB_S fill:#ffcdd2
    style EFUSE_S fill:#ffcdd2
```

---

## 十二、攻击面分析与防御

### 12.1 Secure Boot 攻击分类

```mermaid
graph TB
    subgraph "Secure Boot 攻击面"
        subgraph "物理攻击 Physical"
            PHY1["JTAG/SWD 调试接口<br/>直接读写内存"]
            PHY2["故障注入 Fault Injection<br/>电压毛刺/时钟毛刺"]
            PHY3["侧信道攻击 Side-Channel<br/>功耗/电磁分析"]
            PHY4["芯片拆封/微探针<br/>Decapping/Microprobing"]
        end

        subgraph "软件攻击 Software"
            SW1["Bootloader 漏洞利用<br/>缓冲区溢出/格式串"]
            SW2["签名绕过<br/>验证逻辑缺陷"]
            SW3["TOCTOU 攻击<br/>Time-of-Check-Time-of-Use"]
            SW4["回滚攻击<br/>降级到旧版本"]
        end

        subgraph "供应链攻击 Supply Chain"
            SC1["密钥泄露<br/>签名密钥被盗"]
            SC2["工厂植入<br/>产线篡改镜像"]
            SC3["第三方组件<br/>恶意固件"]
        end

        subgraph "逻辑攻击 Logical"
            LG1["旁路启动<br/>Bypass Boot"]
            LG2["降级刷机<br/>Downgrade Flash"]
            LG3["分区替换<br/>Partition Swap"]
        end
    end

    style PHY1 fill:#ef5350,color:#fff
    style PHY2 fill:#ef5350,color:#fff
    style SW1 fill:#ff8a80
    style SW2 fill:#ff8a80
    style SC1 fill:#ffcdd2
    style LG1 fill:#fff3e0
```

### 12.2 攻击向量与防御措施对应

```mermaid
graph LR
    subgraph "攻击向量"
        A1["JTAG 攻击"]
        A2["故障注入"]
        A3["签名绕过"]
        A4["回滚攻击"]
        A5["密钥泄露"]
        A6["TOCTOU"]
    end

    subgraph "防御措施"
        D1["eFuse 禁用 JTAG<br/>Secure Debug Policy"]
        D2["电压/时钟监测<br/>Glitch Detector"]
        D3["多重签名验证<br/>证书链校验"]
        D4["eFuse/RPMB 计数器<br/>版本单调递增"]
        D5["HSM 离线管理<br/>M-of-N 分片"]
        D6["原子操作<br/>验证后立即执行"]
    end

    A1 -->|防御| D1
    A2 -->|防御| D2
    A3 -->|防御| D3
    A4 -->|防御| D4
    A5 -->|防御| D5
    A6 -->|防御| D6

    style A1 fill:#ef5350,color:#fff
    style A2 fill:#ef5350,color:#fff
    style A3 fill:#ef5350,color:#fff
    style A4 fill:#ef5350,color:#fff
    style A5 fill:#ef5350,color:#fff
    style A6 fill:#ef5350,color:#fff
    style D1 fill:#c8e6c9
    style D2 fill:#c8e6c9
    style D3 fill:#c8e6c9
    style D4 fill:#c8e6c9
    style D5 fill:#c8e6c9
    style D6 fill:#c8e6c9
```

### 12.3 车载特有攻击场景

```mermaid
graph TB
    subgraph "车载 Secure Boot 特有威胁"
        subgraph "OBD-II 接口攻击"
            OBD1["通过 OBD 端口<br/>注入恶意 CAN 消息"]
            OBD2["触发 ECU 固件更新"]
            OBD3["绕过网关刷入恶意固件"]
        end

        subgraph "Infotainment 攻击"
            IVI1["USB/蓝牙/WiFi 入口"]
            IVI2["利用媒体解析漏洞"]
            IVI3["逃逸到系统分区"]
        end

        subgraph "远程攻击 (Connected Car)"
            RMT1["伪造 OTA 更新服务器"]
            RMT2["中间人攻击 OTA 通道"]
            RMT3["云端签名服务入侵"]
        end

        subgraph "车载防御体系"
            DEF1["安全网关隔离"]
            DEF2["Secure Boot 强制验证"]
            DEF3["TLS 双向认证"]
            DEF4["安全 OTA 签名验证"]
        end
    end

    OBD3 -.->|"防御"| DEF1
    IVI3 -.->|"防御"| DEF2
    RMT1 -.->|"防御"| DEF3
    RMT2 -.->|"防御"| DEF4

    style OBD1 fill:#ef5350,color:#fff
    style IVI1 fill:#ff8a80
    style RMT1 fill:#ffcdd2
    style DEF1 fill:#c8e6c9
    style DEF2 fill:#c8e6c9
    style DEF3 fill:#c8e6c9
    style DEF4 fill:#c8e6c9
```

---

## 十三、车载安全标准与合规

### 13.1 相关标准体系

```mermaid
mindmap
  root((车载安全<br/>标准体系))
    功能安全
      ISO 26262
        ASIL A-D
        系统设计
        软件开发
    信息安全
      ISO/SAE 21434
        TARA 威胁分析
        安全开发生命周期
        风险评估
    法规要求
      UNECE WP.29
        R155 网络安全
        R156 软件更新
        型式审批
    行业标准
      AUTOSAR
        SecOC 安全通信
        Crypto Stack
      CC (Common Criteria)
        EAL4+ 评估
        Protection Profile
    平台安全
      ARM PSA
        Secure Boot
        Trusted Firmware
      Google CDD
        Verified Boot 要求
        TEE 要求
```

### 13.2 标准对 Secure Boot 的要求

| 标准 | 对 Secure Boot 的要求 | 等级 |
|------|---------------------|------|
| **UNECE R155** | 车辆需具备安全启动机制，防止未经授权的软件运行 | 法规强制 |
| **UNECE R156** | 软件更新过程需验证更新包完整性和真实性 | 法规强制 |
| **ISO/SAE 21434** | 在威胁分析中识别启动安全风险，制定缓解措施 | 行业要求 |
| **ISO 26262** | 安全关键 ECU 的固件完整性保护（ASIL B-D） | 功能安全 |
| **Google CDD** | Android 设备必须支持 Verified Boot 2.0 | 平台要求 |
| **ARM PSA** | Trusted Firmware 需实现 Secure Boot 链 | 平台推荐 |
| **CC EAL4+** | HSM/TEE 需通过 Common Criteria 评估认证 | 安全评估 |

### 13.3 UNECE R155/R156 与 Secure Boot

```mermaid
graph TB
    subgraph "UNECE R155: 网络安全管理系统 (CSMS)"
        R155_1["威胁识别<br/>识别启动安全威胁"]
        R155_2["风险评估<br/>评估篡改风险等级"]
        R155_3["安全措施<br/>Secure Boot 实施"]
        R155_4["监测响应<br/>异常启动检测"]
        R155_5["型式审批<br/>安全合规证明"]

        R155_1 --> R155_2 --> R155_3 --> R155_4 --> R155_5
    end

    subgraph "UNECE R156: 软件更新管理系统 (SUMS)"
        R156_1["更新验证<br/>OTA 包签名验证"]
        R156_2["回滚保护<br/>防止降级"]
        R156_3["版本管理<br/>RXSWIN 追踪"]
        R156_4["更新安全<br/>端到端加密"]
        R156_5["审批记录<br/>更新历史审计"]

        R156_1 --> R156_2 --> R156_3 --> R156_4 --> R156_5
    end

    R155_3 -.->|"互相关联"| R156_1
    R155_4 -.->|"互相关联"| R156_2

    style R155_3 fill:#ffcdd2
    style R156_1 fill:#ffcdd2
```

---

## 十四、概念发散与关联技术

### 14.1 安全启动相关概念全景

```mermaid
mindmap
  root((Secure Boot<br/>概念全景))
    硬件安全基础
      ARM TrustZone
        Normal World
        Secure World
        SMC 调用
      TPM 可信平台模块
        PCR 平台配置寄存器
        Measured Boot
        Remote Attestation
      HSM 硬件安全模块
        密钥安全存储
        密码运算加速
        FIPS 140-2/3
      SE 安全元件
        JavaCard
        GlobalPlatform
        SIM/eSIM
    密码学基础
      非对称加密
        RSA-2048/4096
        ECDSA P-256/P-384
        EdDSA Ed25519
      哈希算法
        SHA-256
        SHA-384
        SHA3-256
      数字签名
        PKCS#1 v2.1
        ECDSA
        证书链
      PKI 公钥基础设施
        X.509 证书
        CA 证书颁发
        CRL 吊销列表
    启动安全技术
      UEFI Secure Boot
        Secure Variables
        PK/KEK/db/dbx
        Shim Loader
      Measured Boot
        TPM PCR
        Event Log
        IMA 完整性度量
      dm-verity
        Merkle Hash Tree
        按需验证
        只读保护
      AVB 2.0
        vbmeta
        Chained Partition
        Rollback Index
    车载安全扩展
      安全 OTA
        A/B 分区
        差分更新
        增量签名
      ECU 安全启动
        AUTOSAR SecureBoot
        HSM Boot
        CAN 认证
      V2X 安全
        PKI 证书
        SCMS
        假名证书
      安全诊断
        UDS 安全访问
        种子-密钥机制
        ECU 解锁认证
```

### 14.2 UEFI Secure Boot 对比

车载 Android 系统的 Secure Boot 与 PC 端 UEFI Secure Boot 有相似之处但也有显著差异。

```mermaid
graph TB
    subgraph "UEFI Secure Boot (PC)"
        subgraph "密钥层级"
            PK["PK (Platform Key)<br/>平台密钥 (OEM)"]
            KEK["KEK (Key Exchange Key)<br/>密钥交换密钥"]
            DB["db (Signature Database)<br/>允许签名数据库"]
            DBX["dbx (Forbidden Signatures)<br/>禁止签名数据库"]
        end

        subgraph "启动流程"
            UEFI_FW["UEFI Firmware"]
            SHIM["Shim Loader"]
            GRUB["GRUB Bootloader"]
            OS_KERNEL["OS Kernel"]
        end

        PK --> KEK --> DB
        PK --> DBX
        UEFI_FW -->|"验证"| SHIM
        SHIM -->|"验证"| GRUB
        GRUB -->|"验证"| OS_KERNEL
    end

    subgraph "车载 Secure Boot"
        subgraph "密钥层级"
            ROOT["OEM Root Key<br/>根密钥 (eFuse)"]
            PLAT["Platform Key<br/>平台签名密钥"]
            AVB_K["AVB Key<br/>AVB 签名密钥"]
        end

        subgraph "启动流程"
            BROM["BootROM"]
            PBL_B["PBL"]
            SBL_B["SBL/LK"]
            KERN_B["Kernel"]
        end

        ROOT --> PLAT --> AVB_K
        BROM -->|"验证"| PBL_B
        PBL_B -->|"验证"| SBL_B
        SBL_B -->|"验证"| KERN_B
    end

    style PK fill:#e1f5fe
    style ROOT fill:#ffcdd2
```

| 对比维度 | UEFI Secure Boot | 车载 Secure Boot |
|---------|-----------------|-----------------|
| **信任根** | UEFI 固件 + PK | BootROM + eFuse |
| **密钥管理** | 可更新 (db/dbx) | eFuse 不可修改 |
| **灵活性** | 用户可添加签名 | OEM 完全控制 |
| **吊销机制** | dbx 黑名单 | 回滚保护 + OTA |
| **启动模式** | Setup/User/Custom | Locked/Unlocked |
| **标准组织** | UEFI Forum | Google (AVB) + 芯片厂商 |
| **可信度量** | TPM Measured Boot | TEE Boot State |
| **复杂度** | 高 (多变量) | 相对简洁 |

### 14.3 TPM 与 ARM TrustZone 对比

```mermaid
graph TB
    subgraph "TPM (可信平台模块)"
        subgraph "特性"
            TPM1["独立芯片"]
            TPM2["PCR 度量寄存器"]
            TPM3["密封 (Sealing)"]
            TPM4["远程证明"]
        end

        subgraph "启动安全"
            TPM_MB["Measured Boot<br/>记录启动度量值"]
            TPM_SEAL["密钥密封<br/>绑定启动状态"]
        end
    end

    subgraph "ARM TrustZone"
        subgraph "特性"
            TZ1["CPU 内置安全扩展"]
            TZ2["双世界隔离"]
            TZ3["安全中断"]
            TZ4["安全外设"]
        end

        subgraph "启动安全"
            TZ_SB["Secure Boot<br/>验证并执行"]
            TZ_RT["Root of Trust<br/>传递启动状态"]
        end
    end

    style TPM1 fill:#e1f5fe
    style TZ1 fill:#ffcdd2
    style TPM_MB fill:#c8e6c9
    style TZ_SB fill:#c8e6c9
```

| 对比维度 | TPM | ARM TrustZone |
|---------|-----|--------------|
| **形态** | 独立芯片 (离散/集成) | CPU 内置安全扩展 |
| **启动安全方式** | Measured Boot (度量) | Secure Boot (验证) |
| **失败行为** | 记录但不阻断 | 阻断启动 |
| **远程证明** | PCR Quote | Key Attestation |
| **标准** | TCG TPM 2.0 | ARM PSA |
| **车载应用** | 较少 (高通部分使用 fTPM) | 广泛 (主流车载 SoC) |
| **算法支持** | RSA/ECC/SHA/HMAC | 依赖 TEE OS 实现 |
| **代价** | 额外芯片成本 | 零额外硬件成本 |

### 14.4 Measured Boot 与 IMA

```mermaid
graph TB
    subgraph "Measured Boot (度量启动)"
        MB_BIOS["BIOS/UEFI"]
        MB_BL["Bootloader"]
        MB_KERN["Kernel"]
        MB_MOD["Kernel Modules"]

        MB_BIOS -->|"PCR Extend"| PCR0["PCR[0]: Firmware"]
        MB_BL -->|"PCR Extend"| PCR4["PCR[4]: Bootloader"]
        MB_KERN -->|"PCR Extend"| PCR8["PCR[8]: Kernel"]
        MB_MOD -->|"PCR Extend"| PCR9["PCR[9]: Modules"]

        PCR0 --> QUOTE["TPM Quote<br/>签名度量摘要"]
        PCR4 --> QUOTE
        PCR8 --> QUOTE
        PCR9 --> QUOTE
        QUOTE --> VERIFY["远程验证服务器<br/>比对预期值"]
    end

    subgraph "IMA (完整性度量架构)"
        IMA_POLICY["IMA 策略<br/>度量规则"]
        IMA_FILE["文件访问"]
        IMA_HASH["计算文件哈希"]
        IMA_LOG["度量日志<br/>(/sys/kernel/security/ima)"]
        IMA_TPM["PCR Extend<br/>PCR[10]"]
        IMA_APPR["IMA Appraise<br/>签名验证 (可选)"]

        IMA_POLICY --> IMA_FILE --> IMA_HASH
        IMA_HASH --> IMA_LOG
        IMA_HASH --> IMA_TPM
        IMA_HASH --> IMA_APPR
    end

    style PCR0 fill:#e1f5fe
    style PCR4 fill:#e1f5fe
    style PCR8 fill:#e1f5fe
    style PCR9 fill:#e1f5fe
    style IMA_LOG fill:#fff3e0
    style IMA_APPR fill:#c8e6c9
```

### 14.5 AUTOSAR Secure Boot

在传统汽车 ECU 领域，AUTOSAR 定义了自己的 Secure Boot 规范，与 Android Verified Boot 形成互补。

```mermaid
graph TB
    subgraph "AUTOSAR Secure Boot 架构"
        subgraph "Crypto Service Manager (CSM)"
            CSM["CSM<br/>密码服务管理"]
            CSM_VER["验证服务<br/>SignatureVerify"]
            CSM_HASH["哈希服务<br/>HashCompute"]
        end

        subgraph "Crypto Driver (CryIf)"
            CRYIF["Crypto Interface"]
            HSM_DRV["HSM Driver"]
            SW_CRYPTO["Software Crypto"]
        end

        subgraph "Secure Boot 流程"
            STAGE1["Stage 1: HSM Boot<br/>HSM 自验证"]
            STAGE2["Stage 2: Core Boot<br/>验证 Application Core"]
            STAGE3["Stage 3: App Verify<br/>验证应用软件"]
        end

        subgraph "HSM (Hardware Security Module)"
            HSM_HW["汽车级 HSM<br/>(SHE / EVITA)"]
            HSM_KEY["安全密钥存储"]
            HSM_ACC["加密加速器"]
        end
    end

    CSM --> CSM_VER
    CSM --> CSM_HASH
    CSM_VER --> CRYIF
    CRYIF --> HSM_DRV
    CRYIF --> SW_CRYPTO
    HSM_DRV --> HSM_HW

    STAGE1 -->|"HSM 自检"| STAGE2
    STAGE2 -->|"验证 MCU 固件"| STAGE3
    HSM_HW --> STAGE1

    style HSM_HW fill:#ff8a80
    style STAGE1 fill:#ffcdd2
    style STAGE2 fill:#fff3e0
    style STAGE3 fill:#e1f5fe
```

### 14.6 安全启动技术演进时间线

```mermaid
timeline
    title 安全启动技术演进
    section 2000-2010 早期
        2003 : TCG TPM 1.1 规范发布
             : Measured Boot 概念提出
        2006 : ARM TrustZone 技术推出
             : 硬件安全世界隔离
        2007 : UEFI Secure Boot 草案
             : PC 安全启动标准化
    section 2011-2015 标准化
        2011 : UEFI Secure Boot 2.3.1
             : Windows 8 强制要求
        2013 : Android 4.4 Verified Boot v1
             : dm-verity 引入
        2014 : ARM PSA 提出
             : 平台安全架构
    section 2016-2020 成熟期
        2017 : Android 8.0 AVB 2.0 发布
             : vbmeta + 链式分区
        2018 : Android 9.0 强制 AVB
             : Rollback Protection 强化
        2019 : AUTOSAR Secure Boot
             : 车规级安全启动
        2020 : UNECE WP.29 R155/R156
             : 车辆网络安全法规
    section 2021-2025 深化期
        2021 : ISO/SAE 21434 发布
             : 汽车网络安全工程
        2022 : Android 13 KeyMint
             : 硬件级密钥管理强化
        2023 : UNECE R155/R156 强制执行
             : 欧洲新车型式审批要求
        2024 : Automotive Grade Linux
             : 开源车载安全启动方案
        2025 : 车载 SoC Secure Boot
             : 多域融合安全启动
```

### 14.7 数字签名原理深入

数字签名是 Secure Boot 的密码学基础，理解签名原理对于理解整个安全启动机制至关重要。

```mermaid
graph TB
    subgraph "签名流程 (OEM 端)"
        ORIG["原始镜像<br/>boot.img"]
        HASH_S["SHA-256 哈希"]
        PRIV_KEY["OEM 私钥<br/>(HSM 内)"]
        SIGN_OP["RSA/ECDSA<br/>签名运算"]
        SIGNED["签名后镜像<br/>boot.img + 签名"]

        ORIG --> HASH_S
        HASH_S --> SIGN_OP
        PRIV_KEY --> SIGN_OP
        SIGN_OP --> SIGNED
    end

    subgraph "验证流程 (设备端)"
        RECV["接收到的镜像"]
        SPLIT_D["镜像数据"]
        SPLIT_S["签名数据"]
        HASH_V["SHA-256 哈希"]
        PUB_KEY["OEM 公钥<br/>(eFuse/证书)"]
        VERIFY_OP["RSA/ECDSA<br/>验证运算"]
        COMPARE["比对哈希值"]
        RESULT{"匹配？"}
        OK["✓ 验证通过"]
        FAIL["✗ 验证失败"]

        RECV --> SPLIT_D
        RECV --> SPLIT_S
        SPLIT_D --> HASH_V
        SPLIT_S --> VERIFY_OP
        PUB_KEY --> VERIFY_OP
        VERIFY_OP --> COMPARE
        HASH_V --> COMPARE
        COMPARE --> RESULT
        RESULT -->|Yes| OK
        RESULT -->|No| FAIL
    end

    SIGNED -.->|"传输/存储"| RECV

    style PRIV_KEY fill:#ff8a80
    style PUB_KEY fill:#c8e6c9
    style OK fill:#c8e6c9
    style FAIL fill:#ef5350,color:#fff
```

### 14.8 证书链验证

```mermaid
graph TB
    subgraph "Secure Boot 证书链"
        subgraph "Root CA (离线)"
            ROOT_CA["Root CA Certificate<br/>自签名根证书<br/>有效期: 30 年"]
            ROOT_PK["Root Private Key<br/>离线 HSM 保管"]
        end

        subgraph "Intermediate CA"
            INT_CA["Intermediate CA Certificate<br/>中间 CA 证书<br/>有效期: 10 年"]
            INT_PK["Intermediate Private Key<br/>在线 HSM 保管"]
        end

        subgraph "Signing Certificate"
            SIGN_CERT["Signing Certificate<br/>签名证书<br/>有效期: 3 年"]
            SIGN_PK["Signing Private Key<br/>签名服务器"]
        end

        subgraph "设备端验证"
            DEV_ROOT["预置 Root CA 公钥<br/>(eFuse Hash)"]
            VERIFY_1["验证 Intermediate CA"]
            VERIFY_2["验证 Signing Cert"]
            VERIFY_3["验证镜像签名"]
        end
    end

    ROOT_PK -->|"签署"| INT_CA
    INT_PK -->|"签署"| SIGN_CERT
    SIGN_PK -->|"签署"| VERIFY_3

    DEV_ROOT -->|"信任锚"| VERIFY_1
    VERIFY_1 --> VERIFY_2
    VERIFY_2 --> VERIFY_3

    style ROOT_CA fill:#ff8a80
    style INT_CA fill:#ffcdd2
    style SIGN_CERT fill:#fff3e0
    style DEV_ROOT fill:#c8e6c9
```

---

## 十五、最佳实践与实施建议

### 15.1 车载 Secure Boot 实施检查清单

```mermaid
graph TB
    subgraph "实施检查清单"
        subgraph "Phase 1: 设计阶段"
            CK1["☐ 定义信任链架构"]
            CK2["☐ 选择密码学算法"]
            CK3["☐ 设计密钥层级"]
            CK4["☐ 制定回滚保护策略"]
            CK5["☐ 规划 eFuse 位分配"]
        end

        subgraph "Phase 2: 开发阶段"
            CK6["☐ 搭建签名基础设施"]
            CK7["☐ 实现签名验证逻辑"]
            CK8["☐ 集成 AVB 2.0"]
            CK9["☐ 实现 dm-verity"]
            CK10["☐ 对接 TEE Boot State"]
        end

        subgraph "Phase 3: 测试阶段"
            CK11["☐ 正常启动验证测试"]
            CK12["☐ 篡改镜像拒绝测试"]
            CK13["☐ 回滚攻击防御测试"]
            CK14["☐ 故障恢复测试"]
            CK15["☐ 性能影响评估"]
        end

        subgraph "Phase 4: 生产阶段"
            CK16["☐ 产线 eFuse 烧写流程"]
            CK17["☐ 密钥仪式执行"]
            CK18["☐ 签名流水线部署"]
            CK19["☐ OTA 更新流程验证"]
            CK20["☐ 安全审计与合规"]
        end
    end

    CK5 --> CK6
    CK10 --> CK11
    CK15 --> CK16

    style CK1 fill:#e1f5fe
    style CK6 fill:#fff3e0
    style CK11 fill:#c8e6c9
    style CK16 fill:#ffcdd2
```

### 15.2 关键安全建议

**密钥管理：**

- 根密钥必须在离线 HSM 中生成和存储，采用 M-of-N 分片方案（如 3-of-5）
- 签名密钥定期轮换（建议每年一次），支持多代密钥共存过渡期
- 严禁在开发/测试环境中使用生产密钥，必须使用独立的测试密钥体系
- 密钥仪式（Key Ceremony）需多人见证并全程录像存档

**eFuse 规划：**

- 预留充足的 Anti-Rollback Counter 位宽（建议 32 bit 以上）
- Secure Boot Enable 位一旦烧写不可回退，需在产线严格控制
- JTAG Disable 和 DAA Disable 应在产品量产前烧写
- 保留调试 Fuse 用于售后返修（通过安全认证机制控制）

**验证策略：**

- 车载安全关键域（仪表/ADAS）建议采用验证失败直接停止启动的强制策略
- IVI 娱乐域可采用验证失败重启或降级运行策略
- 所有验证失败事件必须记录审计日志
- dm-verity 错误策略根据分区安全等级差异化配置

**OTA 安全：**

- OTA 更新包必须使用独立的 OTA 签名密钥，与 Secure Boot 签名密钥分离
- 支持 A/B 分区无缝更新，确保更新失败可安全回退
- 更新后首次启动成功标记（Mark Successful）前不得更新回滚计数器
- OTA 通道必须使用 TLS 1.3 + 双向证书认证

### 15.3 Secure Boot 启动性能优化

```mermaid
graph LR
    subgraph "性能优化策略"
        subgraph "算法选择"
            OPT1["ECC P-256 替代 RSA-2048<br/>签名更小、验证更快"]
            OPT2["SHA-256 硬件加速<br/>利用 SoC Crypto Engine"]
        end

        subgraph "并行化"
            OPT3["并行加载 & 验证<br/>加载下一级同时验证当前"]
            OPT4["多核并行哈希<br/>大分区分块计算"]
        end

        subgraph "缓存策略"
            OPT5["dm-verity Hash Cache<br/>已验证块缓存"]
            OPT6["Lazy 验证<br/>按需验证而非全量"]
        end
    end

    style OPT1 fill:#c8e6c9
    style OPT3 fill:#c8e6c9
    style OPT5 fill:#c8e6c9
```

| 优化手段 | 效果 | 适用阶段 |
|---------|------|---------|
| **ECC 替代 RSA** | 签名验证速度提升 3-5x | Bootloader 验证 |
| **SHA 硬件加速** | 哈希计算速度提升 10x+ | 全阶段 |
| **并行加载验证** | 减少等待时间 | Bootloader → Kernel |
| **dm-verity 缓存** | 减少重复 I/O | 运行时 |
| **Lazy Verification** | 按需验证，减少启动耗时 | dm-verity 大分区 |
| **A/B 预验证** | 空闲时预验证备用 Slot | OTA 更新后 |

---

## 附录：术语表

| 术语 | 全称 | 说明 |
|------|------|------|
| **AVB** | Android Verified Boot | Android 验证启动框架 |
| **PBL** | Primary Boot Loader | 一级引导程序 |
| **SBL** | Secondary Boot Loader | 二级引导程序 |
| **LK** | Little Kernel | MTK 平台 Bootloader |
| **ABL** | Android Boot Loader | 高通平台 Bootloader |
| **XBL** | eXtensible Boot Loader | 高通可扩展引导加载器 |
| **OTP** | One-Time Programmable | 一次性可编程 |
| **eFuse** | Electronic Fuse | 电子熔丝 |
| **QFPROM** | Qualcomm Fuse Programming ROM | 高通熔丝编程 ROM |
| **RPMB** | Replay Protected Memory Block | 重放保护存储块 |
| **RoT** | Root of Trust | 信任根 |
| **VB State** | Verified Boot State | 验证启动状态 |
| **dm-verity** | Device-Mapper Verity | 设备映射器完整性验证 |
| **vbmeta** | Verified Boot Metadata | 验证启动元数据 |
| **TEE** | Trusted Execution Environment | 可信执行环境 |
| **QTEE** | Qualcomm TEE | 高通可信执行环境 |
| **HSM** | Hardware Security Module | 硬件安全模块 |
| **TOCTOU** | Time-of-Check-Time-of-Use | 检查时间与使用时间差异攻击 |
| **CSMS** | Cybersecurity Management System | 网络安全管理系统 |
| **SUMS** | Software Update Management System | 软件更新管理系统 |
| **ASIL** | Automotive Safety Integrity Level | 汽车安全完整性等级 |
| **TARA** | Threat Analysis and Risk Assessment | 威胁分析与风险评估 |
