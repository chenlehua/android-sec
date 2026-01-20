# 车载系统安全性提升方案 - MTK平台

基于MTK平台车载系统架构特点，从防护类（兼容备份、仲裁）角度提出针对性安全增强方案。本文档专注于MTK平台（Trustonic TEE）架构。

---

## 目录

1. [MTK平台架构分析](#一mtk平台架构分析)
2. [现有安全能力评估](#二现有安全能力评估)
3. [TEE单点风险识别与解决方案](#三tee单点风险识别与解决方案)
4. [防护类安全增强方案](#四防护类安全增强方案)
5. [密钥备份与恢复机制](#五密钥备份与恢复机制)
6. [Keystore功能集成方案](#六keystore功能集成方案)
7. [安全仲裁架构设计](#七安全仲裁架构设计)
8. [故障检测与降级策略](#八故障检测与降级策略)
9. [安全监控与审计增强](#九安全监控与审计增强)
10. [实施建议](#十实施建议)

---

## 一、MTK平台架构分析

### 1.1 MTK平台系统架构总览

MTK平台车载系统采用分层架构设计，以Trustonic TEE作为核心安全锚点，从应用层到安全硬件层形成完整的安全链路。

```mermaid
graph TB
    subgraph "应用层 Application Layer"
        APP["SafeKeyManager<br/>Framework封装"]
        APP_DESC["• 证书获取、签名验签<br/>• OkHttpClient TLS配置<br/>• 业务应用安全调用"]
    end

    subgraph "Native层 SafeKeyService"
        NATIVE["SafeKeyService<br/>Native服务"]
        NATIVE_DESC["• 密码学算法封装<br/>• 产线密钥/证书灌装接口<br/>• 安全策略管理"]
    end

    subgraph "中间件层 Middleware"
        MW["安全中间件"]
        SDF["SDF接口<br/>基于GMT 0018"]
        OSSL["OpenSSL Engine"]
        MW_DESC["• 密码运算调度管理<br/>• 密码设备兼容"]
    end

    subgraph "安全硬件层 Security Hardware"
        TEE["Trustonic TEE<br/>Kinibi<br/>安全锚点"]
        TA["Trusted Applications<br/>• Keymaster TA<br/>• Crypto TA<br/>• Storage TA"]
    end

    APP --> NATIVE
    NATIVE --> MW
    SDF --> TEE
    OSSL --> TEE
    TEE --> TA

    style APP fill:#e1f5fe
    style NATIVE fill:#fff3e0
    style MW fill:#f3e5f5
    style TEE fill:#ffcdd2
```

### 1.2 Trustonic TEE架构详解

MTK平台采用Trustonic Kinibi作为TEE实现，基于ARM TrustZone技术提供硬件级安全隔离。

```mermaid
graph TB
    subgraph "MTK平台 Trustonic TEE架构"
        subgraph "普通世界 Normal World"
            ANDROID["Android OS"]
            CA["Client Applications"]
            TEE_CLIENT["TEE Client API"]
        end

        subgraph "安全世界 Secure World"
            KINIBI["Trustonic Kinibi TEE OS"]
            subgraph "Trusted Applications"
                KEYMASTER["Keymaster TA"]
                CRYPTO_TA["Crypto TA"]
                STORAGE_TA["Storage TA"]
                CUSTOM_TA["自定义安全TA"]
            end
        end

        subgraph "硬件层"
            TRUSTZONE["ARM TrustZone"]
            RPMB["RPMB安全存储"]
            EFUSE["eFuse安全熔丝"]
        end
    end

    CA --> TEE_CLIENT
    TEE_CLIENT --> KINIBI
    KINIBI --> KEYMASTER
    KINIBI --> CRYPTO_TA
    KINIBI --> STORAGE_TA
    KINIBI --> CUSTOM_TA
    KINIBI --> TRUSTZONE
    STORAGE_TA --> RPMB

    style KINIBI fill:#ffcdd2
    style TRUSTZONE fill:#ff8a80
```

### 1.3 TEE密码模块功能架构

TEE密码模块作为MTK平台的核心安全锚点，提供完整的密码学服务能力。

```mermaid
graph TB
    subgraph "TEE密码模块 TT模块"
        subgraph "设备与会话管理"
            DEV["设备管理<br/>• 设备SN<br/>• 版本管理"]
            SESS["会话管理<br/>• 多进程支持<br/>• 多线程支持"]
        end

        subgraph "安全与授权"
            AUTH["授权管理<br/>• 身份验证<br/>• 防暴力破解"]
            KEY["密钥管理<br/>• 产线灌装<br/>• 访问控制"]
        end

        subgraph "密码服务"
            CRYPTO["密码服务<br/>• RSA/ECC/SM2<br/>• AES/SM4<br/>• SM3/SHA256"]
            STORE["安全存储<br/>• RPMB存储<br/>• 数据加密<br/>• 完整性保护"]
        end

        subgraph "证书与日志"
            CERT["证书管理<br/>• P10证书请求<br/>• 证书存取"]
            LOG["日志管理<br/>• 等级控制<br/>• 审计日志"]
        end
    end

    subgraph "产线灌装密钥材料"
        KEYS["• TLS证书及公私钥对（双向认证）<br/>• 对称密钥（数据加密）<br/>• 设备唯一标识密钥"]
    end

    DEV --> AUTH
    SESS --> KEY
    AUTH --> CRYPTO
    KEY --> STORE
    CRYPTO --> CERT
    STORE --> LOG
    TEE密码模块 --> KEYS

    style TEE密码模块 fill:#ffebee
```

---

## 二、现有安全能力评估

### 2.1 MTK平台安全能力矩阵

| 能力维度 | 当前状态 | 安全等级 | 说明 |
|---------|---------|---------|------|
| 密钥存储 | TEE安全存储 + RPMB | 高 | 硬件隔离保护 |
| 密码运算 | TEE内执行 | 高 | 密钥不出TEE |
| 身份认证 | 授权管理 + 防暴力破解 | 中高 | 有基础防护 |
| 冗余备份 | **无** | 低 | 单点故障风险 |
| 故障转移 | **无** | 低 | 无降级机制 |
| 安全仲裁 | **无** | 低 | 无分级策略 |
| 密钥恢复 | 仅产线重灌 | 低 | 恢复代价高 |
| 物理防护 | 中等 | 中 | 依赖TrustZone |

### 2.2 MTK平台安全特性

```mermaid
graph LR
    subgraph "MTK平台安全能力"
        M1["✓ ARM TrustZone隔离"]
        M2["✓ RPMB安全存储"]
        M3["✓ 密码运算保护"]
        M4["✓ Trustonic认证"]
        M5["✗ 无独立安全芯片"]
        M6["✗ 物理攻击防护较弱"]
        M7["✗ 无双安全锚点冗余"]
    end
```

### 2.3 架构约束分析

```mermaid
mindmap
  root((MTK TEE-Only架构<br/>约束分析))
    优势
      成本低：无需额外安全芯片
      性能好：TEE运算速度快
      集成度高：与主SoC一体化
      Trustonic成熟稳定
    约束
      单点故障：TEE异常无备用
      物理安全：抗攻击能力弱于SE
      密钥备份：产线灌装难以备份
      故障恢复：损坏需返厂重灌
    设计原则
      最大化TEE约束下的安全性
      通过软件架构弥补硬件冗余
      建立分级防护和智能降级
      强化监控预警提前发现问题
```

### 2.4 MTK平台密钥管理架构

```mermaid
flowchart TB
    subgraph "MTK平台密钥管理"
        direction TB
        MTK_ROOT["设备根密钥<br/>Device Root Key"]
        MTK_TLS["TLS密钥对"]
        MTK_SYM["对称加密密钥"]
        MTK_STORE["Trustonic TEE<br/>RPMB安全存储"]
        
        MTK_ROOT --> MTK_STORE
        MTK_TLS --> MTK_STORE
        MTK_SYM --> MTK_STORE
    end

    style MTK_STORE fill:#ffcdd2
```

---

## 三、TEE单点风险识别与解决方案

### 3.1 MTK平台特有风险

由于MTK平台仅依赖Trustonic TEE作为唯一安全锚点，TEE故障将导致整个安全体系失效。

```mermaid
graph TB
    subgraph "MTK平台TEE故障风险场景"
        subgraph "场景1: TEE软件异常"
            S1_PERF["表现: TA崩溃/响应超时/错误码"]
            S1_PROB["概率: 中等"]
            S1_IMPACT["影响: TLS通信中断/密码服务不可用"]
            S1_RECOVER["恢复: 重启TEE模块/系统重启"]
        end

        subgraph "场景2: TEE安全存储损坏"
            S2_PERF["表现: 密钥读取失败/数据校验错误"]
            S2_PROB["概率: 低"]
            S2_IMPACT["影响: 依赖该密钥的功能失效"]
            S2_RECOVER["恢复: 密钥恢复机制/返厂重灌"]
        end

        subgraph "场景3: RPMB存储故障"
            S3_PERF["表现: 安全存储读写失败"]
            S3_PROB["概率: 极低"]
            S3_IMPACT["影响: 密钥丢失/系统安全状态不可恢复"]
            S3_RECOVER["恢复: 硬件更换"]
        end

        subgraph "场景4: 密钥泄露风险"
            S4_PERF["表现: 侧信道攻击/非法导出"]
            S4_PROB["概率: 极低（TEE保护）"]
            S4_IMPACT["影响: 设备身份冒用/通信窃听"]
            S4_RECOVER["恢复: 证书吊销 + 密钥轮换"]
        end

        subgraph "场景5: 版本回退攻击"
            S5_PERF["表现: 降级到有漏洞的旧版本"]
            S5_PROB["概率: 低"]
            S5_IMPACT["影响: 绕过安全补丁"]
            S5_RECOVER["防护: 防回退机制"]
        end
    end
```

### 3.2 TEE故障风险解决方案

针对MTK平台（Trustonic TEE单安全锚点架构）的各类TEE故障风险场景，提出以下针对性解决方案：

#### 3.2.1 场景1: TEE软件异常解决方案

| 故障表现 | 解决方案 | 实施细节 |
|---------|---------|---------|
| **TA崩溃** | TEE热重启 | 自动重启TEE服务，重新加载TA，期间缓存请求 |
| **响应超时** | 超时重试+降级 | 设置超时阈值（如3秒），超时后重试2次，仍失败则进入降级模式 |
| **错误码返回** | 错误分类处理 | 根据错误码分类：临时错误重试、永久错误降级、严重错误告警 |

```mermaid
flowchart TD
    TEE_SW_FAULT["TEE软件异常检测"]
    TEE_SW_FAULT --> CLASSIFY{"错误分类"}
    
    CLASSIFY -->|临时错误| RETRY["重试操作<br/>最多3次"]
    CLASSIFY -->|永久错误| DEGRADE["进入降级模式"]
    CLASSIFY -->|严重错误| ALERT["立即告警+安全模式"]
    
    RETRY -->|成功| NORMAL["恢复正常"]
    RETRY -->|失败| RESTART["尝试热重启TEE"]
    
    RESTART -->|成功| NORMAL
    RESTART -->|失败| DEGRADE
    
    DEGRADE --> SW_BACKUP["软件密码备份<br/>仅限中低安全级别"]
    DEGRADE --> NOTIFY["通知用户+上报VSOC"]
    
    style SW_BACKUP fill:#fff9c4
    style ALERT fill:#ff8a80
```

**具体措施**：
1. **心跳监测增强**：将心跳检测周期从5秒缩短到2秒，快速发现异常
2. **TA隔离运行**：关键TA独立运行，一个TA崩溃不影响其他TA
3. **状态持久化**：TEE操作状态定期保存，重启后可快速恢复

#### 3.2.2 场景2: TEE安全存储损坏解决方案

| 故障表现 | 解决方案 | 实施细节 |
|---------|---------|---------|
| **密钥读取失败** | 密钥分片恢复 | 从云端和本地分片恢复设备根密钥 |
| **数据校验错误** | 完整性修复 | 尝试从备份区恢复，失败则触发密钥恢复流程 |
| **单个密钥损坏** | 密钥重派生 | 使用根密钥重新派生业务密钥 |

```mermaid
flowchart TD
    STORAGE_FAULT["安全存储损坏检测"]
    STORAGE_FAULT --> CHECK_TYPE{"损坏类型"}
    
    CHECK_TYPE -->|单个密钥| REDERIVE["使用根密钥重新派生"]
    CHECK_TYPE -->|根密钥损坏| SHARD_RECOVER["分片恢复流程"]
    CHECK_TYPE -->|RPMB完全损坏| FACTORY["返厂维修"]
    
    REDERIVE --> VERIFY["验证派生结果"]
    VERIFY -->|成功| NORMAL["恢复正常"]
    VERIFY -->|失败| SHARD_RECOVER
    
    SHARD_RECOVER --> COLLECT["收集密钥分片<br/>本地+云端+4S店"]
    COLLECT --> REBUILD["TEE内重建根密钥"]
    REBUILD --> CERT_RESIGN["证书重签发"]
    CERT_RESIGN --> NORMAL
    
    style SHARD_RECOVER fill:#fff9c4
    style FACTORY fill:#ff8a80
```

**具体措施**：
1. **双区备份**：RPMB存储采用双区设计，主区损坏时自动切换备用区
2. **定期校验**：每小时进行一次密钥完整性校验，提前发现问题
3. **分片预取**：预先将一个密钥分片安全缓存在本地加密分区

#### 3.2.3 场景3: RPMB存储故障解决方案

| 故障表现 | 解决方案 | 实施细节 |
|---------|---------|---------|
| **读写失败** | 多次重试+告警 | 重试读写操作，同时记录错误日志并告警 |
| **完全不可访问** | 安全模式+维修 | 进入安全模式，禁用网联功能，引导用户维修 |
| **部分扇区损坏** | 坏块管理+迁移 | 标记坏块，将数据迁移到备用扇区 |

```mermaid
flowchart TD
    RPMB_FAULT["RPMB故障检测"]
    RPMB_FAULT --> SEVERITY{"故障严重程度"}
    
    SEVERITY -->|轻微| BAD_BLOCK["坏块标记与迁移"]
    SEVERITY -->|中等| RETRY_WARN["重试+持续告警"]
    SEVERITY -->|严重| SAFE_MODE["进入安全模式"]
    
    BAD_BLOCK --> CONTINUE["继续运行<br/>监控告警"]
    
    RETRY_WARN -->|恢复| CONTINUE
    RETRY_WARN -->|持续失败| SAFE_MODE
    
    SAFE_MODE --> DISABLE["禁用网联功能"]
    SAFE_MODE --> LOCAL_ONLY["仅保留基本车辆功能"]
    SAFE_MODE --> USER_NOTIFY["强制显示维修提示"]
    
    USER_NOTIFY --> FACTORY["引导用户前往4S店"]
    
    style SAFE_MODE fill:#ff8a80
    style FACTORY fill:#ffcdd2
```

**具体措施**：
1. **RPMB健康监测**：启动时和定期检测RPMB健康状态
2. **写入验证**：每次写入后立即读取验证，确保写入成功
3. **备用存储预案**：准备临时的软件加密存储作为紧急备用

#### 3.2.4 场景4: 密钥泄露风险解决方案

| 风险类型 | 解决方案 | 实施细节 |
|---------|---------|---------|
| **侧信道攻击风险** | TEE安全加固 | 启用Trustonic的侧信道防护，恒定时间实现 |
| **非法导出尝试** | 访问审计+阻断 | 记录所有密钥访问，检测异常模式立即阻断 |
| **密钥已泄露** | 证书吊销+轮换 | 远程吊销证书，触发密钥轮换流程 |

```mermaid
flowchart TD
    LEAK_RISK["密钥泄露风险检测"]
    LEAK_RISK --> TYPE{"检测类型"}
    
    TYPE -->|预防性检测| MONITOR["异常访问监控"]
    TYPE -->|确认泄露| REVOKE["立即吊销证书"]
    
    MONITOR --> PATTERN["行为模式分析"]
    PATTERN -->|正常| CONTINUE["继续监控"]
    PATTERN -->|异常| LOCKDOWN["临时锁定密钥"]
    
    LOCKDOWN --> INVESTIGATE["安全调查"]
    INVESTIGATE -->|误报| UNLOCK["解锁密钥"]
    INVESTIGATE -->|确认泄露| REVOKE
    
    REVOKE --> KEY_ROTATE["密钥轮换"]
    KEY_ROTATE --> CERT_REISSUE["证书重签发"]
    CERT_REISSUE --> VSOC_REPORT["上报安全事件"]
    
    style REVOKE fill:#ff8a80
    style KEY_ROTATE fill:#fff9c4
```

**具体措施**：
1. **访问频率限制**：限制密钥访问频率，防止暴力攻击
2. **异常行为检测**：检测非常规时间、非常规调用者的访问
3. **远程吊销能力**：车厂可远程吊销设备证书，使泄露密钥失效

#### 3.2.5 场景5: 版本回退攻击解决方案

| 攻击类型 | 解决方案 | 实施细节 |
|---------|---------|---------|
| **TEE OS回退** | Anti-Rollback计数器 | 使用eFuse存储版本号，禁止回退到低版本 |
| **TA回退** | TA版本绑定 | TA版本与设备状态绑定，回退时密钥不可用 |
| **固件回退** | Verified Boot | 启动时验证固件签名和版本号 |

```mermaid
flowchart TD
    ROLLBACK["版本回退检测"]
    ROLLBACK --> CHECK{"版本比对"}
    
    CHECK -->|版本正常| BOOT["正常启动"]
    CHECK -->|版本回退| BLOCK["阻止启动"]
    
    BLOCK --> LOG["记录回退尝试"]
    LOG --> ALERT["安全告警"]
    ALERT --> FORCE_UPDATE["强制更新到最新版本"]
    
    subgraph "Anti-Rollback机制"
        AR1["eFuse版本计数器"]
        AR2["每次升级递增"]
        AR3["只增不减"]
    end
    
    style BLOCK fill:#ff8a80
    style ALERT fill:#ffcdd2
```

**具体措施**：
1. **eFuse版本锁定**：关键安全组件版本记录在eFuse，不可逆
2. **双重版本验证**：Bootloader和TEE均进行版本验证
3. **升级审计**：记录每次升级事件，便于追溯

### 3.3 故障影响链分析

```mermaid
flowchart TD
    ROOT["Trustonic TEE密码模块故障"]
    
    ROOT --> TLS["TLS证书/私钥不可用"]
    ROOT --> SYM["对称密钥不可用"]
    ROOT --> SERVICE["密码服务不可用"]
    
    TLS --> TLS1["与云端TLS通信失败"]
    TLS --> TLS2["OTA更新签名验证失败"]
    TLS --> TLS3["远程诊断功能失效"]
    
    SYM --> SYM1["本地数据解密失败"]
    SYM --> SYM2["安全存储不可访问"]
    SYM --> SYM3["用户隐私数据无法读取"]
    
    SERVICE --> SVC1["域间安全通信中断"]
    SERVICE --> SVC2["签名验签功能失效"]
    SERVICE --> SVC3["安全认证功能失效"]
    
    TLS1 --> FINAL["最终影响: 车辆网联功能全面降级<br/>需返厂维修"]
    TLS2 --> FINAL
    TLS3 --> FINAL
    SYM1 --> FINAL
    SYM2 --> FINAL
    SYM3 --> FINAL
    SVC1 --> FINAL
    SVC2 --> FINAL
    SVC3 --> FINAL

    style ROOT fill:#ff5252
    style FINAL fill:#ff8a80
```

### 3.4 MTK平台风险缓解重点

由于缺少独立安全芯片SE作为备份，MTK平台需要更加重视：

1. **TEE健康监测**：更频繁的心跳检测和功能自检
2. **密钥备份机制**：完善的分片备份和恢复流程
3. **软件降级方案**：中低安全级别操作的软件备份
4. **预警机制**：提前发现问题，避免故障发生

---

## 四、防护类安全增强方案

### 4.1 MTK平台防护架构

```mermaid
graph TB
    subgraph "MTK平台TEE-Only架构防护增强方案"
        subgraph "安全策略管理层 新增"
            POL1["安全仲裁引擎"]
            POL2["降级策略管理"]
            POL3["安全状态监控"]
        end

        subgraph "密钥保护增强层 新增"
            KEY1["密钥分片备份<br/>云端+本地"]
            KEY2["密钥版本管理<br/>轮换+派生"]
            KEY3["密钥恢复服务<br/>多因素验证"]
        end

        subgraph "健康监测与预警层 新增"
            MON1["TEE健康检测"]
            MON2["异常行为检测"]
            MON3["预警与告警"]
        end

        subgraph "中间件层增强"
            MW1["SDF接口"]
            MW2["OpenSSL Engine"]
            MW3["软件密码备份模块<br/>新增"]
        end

        subgraph "安全硬件层"
            TEE["Trustonic TEE<br/>密码模块"]
        end
    end

    POL1 --> KEY1
    POL2 --> KEY2
    POL3 --> KEY3
    KEY1 --> MON1
    KEY2 --> MON2
    KEY3 --> MON3
    MON1 --> MW1
    MON2 --> MW2
    MON3 --> MW3
    MW1 --> TEE
    MW2 --> TEE

    style POL1 fill:#c8e6c9
    style KEY1 fill:#c8e6c9
    style MON1 fill:#c8e6c9
    style MW3 fill:#c8e6c9
```

### 4.2 分层防护策略

```mermaid
graph TB
    subgraph "第一层: 预防性防护"
        L1_1["TEE健康监测: 定期心跳检测、性能监控"]
        L1_2["密钥完整性校验: 启动时和定期校验密钥哈希"]
        L1_3["防回退保护: 版本号递增验证"]
        L1_4["访问频率监控: 异常调用检测"]
    end

    subgraph "第二层: 检测性防护"
        L2_1["操作结果验证: 密码操作结果完整性检查"]
        L2_2["异常模式识别: 基于规则的异常行为检测"]
        L2_3["安全审计日志: 关键操作全记录"]
        L2_4["实时告警: 严重异常即时上报"]
    end

    subgraph "第三层: 响应性防护"
        L3_1["智能降级: 根据故障类型自动切换安全模式"]
        L3_2["功能限制: 禁用高风险远程功能"]
        L3_3["密钥恢复: 触发密钥恢复流程"]
        L3_4["用户通知: 告知用户系统状态"]
    end

    subgraph "第四层: 恢复性防护"
        L4_1["密钥分片恢复: 从多方收集分片重建密钥"]
        L4_2["证书重签发: 通过安全通道申请新证书"]
        L4_3["系统重置: 必要时恢复到安全初始状态"]
        L4_4["返厂重灌: 最后手段，产线重新灌装"]
    end

    L1_1 --> L2_1
    L1_2 --> L2_2
    L1_3 --> L2_3
    L1_4 --> L2_4
    L2_1 --> L3_1
    L2_2 --> L3_2
    L2_3 --> L3_3
    L2_4 --> L3_4
    L3_1 --> L4_1
    L3_2 --> L4_2
    L3_3 --> L4_3
    L3_4 --> L4_4

    style L1_1 fill:#e8f5e9
    style L2_1 fill:#fff3e0
    style L3_1 fill:#fff9c4
    style L4_1 fill:#ffebee
```

---

## 五、密钥备份与恢复机制

### 5.1 密钥分类与备份策略

```mermaid
graph TB
    subgraph "密钥分类与备份策略矩阵"
        subgraph "设备根密钥 Device Root Key"
            DRK_LEVEL["安全级别: 最高（产线灌装）"]
            DRK_BACKUP["备份策略: 分片云端托管 + 本地加密"]
            DRK_RECOVER["恢复方式: 多因素验证恢复 + 车厂授权"]
        end

        subgraph "TLS证书私钥"
            TLS_LEVEL["安全级别: 高"]
            TLS_BACKUP["备份策略: 加密备份（需要根密钥）"]
            TLS_RECOVER["恢复方式: 证书重签发（云端+本地）"]
        end

        subgraph "业务对称密钥"
            SYM_LEVEL["安全级别: 高（产线灌装）"]
            SYM_BACKUP["备份策略: 派生自根密钥（无需备份）"]
            SYM_RECOVER["恢复方式: 重新派生（根密钥+种子）"]
        end

        subgraph "会话密钥"
            SESS_LEVEL["安全级别: 中（短期有效）"]
            SESS_BACKUP["备份策略: 不备份（易重建）"]
            SESS_RECOVER["恢复方式: 重新协商"]
        end

        subgraph "用户数据加密密钥"
            USER_LEVEL["安全级别: 中"]
            USER_BACKUP["备份策略: 派生自业务密钥（无需备份）"]
            USER_RECOVER["恢复方式: 重新派生"]
        end
    end
```

### 5.2 密钥派生架构

通过基于根密钥的派生架构，大幅减少需要备份的密钥数量。

```mermaid
flowchart TD
    DRK["设备根密钥 DRK<br/>256-bit AES<br/>产线灌装，唯一需要备份"]

    DRK --> CMK["通信主密钥 CMK<br/>KDF(DRK, 'COMM')"]
    DRK --> SMK["存储主密钥 SMK<br/>KDF(DRK, 'STOR')"]
    DRK --> VMK["签名主密钥 VMK<br/>KDF(DRK, 'SIGN')"]

    CMK --> TLS_KEY["TLS密钥"]
    CMK --> SESS_KEY["会话密钥"]
    
    SMK --> USER_DATA["用户数据加密"]
    SMK --> SYS_CONFIG["系统配置加密"]
    
    VMK --> OTA_KEY["OTA验签密钥"]
    VMK --> LOG_KEY["日志签名密钥"]

    subgraph "派生优势"
        ADV1["• 只需备份设备根密钥"]
        ADV2["• 其他密钥可重新派生"]
        ADV3["• 减少密钥暴露面"]
        ADV4["• 支持密钥轮换"]
    end

    style DRK fill:#ff8a80
    style CMK fill:#ffcc80
    style SMK fill:#ffcc80
    style VMK fill:#ffcc80
```

### 5.3 设备根密钥分片备份方案

采用Shamir秘密分享方案，实现安全的密钥备份与恢复。

```mermaid
flowchart TD
    DRK["设备根密钥 DRK<br/>256-bit"]
    
    DRK --> SHAMIR["Shamir分片算法<br/>n=4, k=3"]
    
    SHAMIR --> S1["分片 1<br/>本地RPMB备份<br/>设备唯一标识加密"]
    SHAMIR --> S2["分片 2<br/>车厂云端安全存储<br/>设备认证获取"]
    SHAMIR --> S3["分片 3<br/>第三方托管HSM<br/>可选增强"]
    SHAMIR --> S4["分片 4<br/>离线冷备份<br/>4S店存储"]

    subgraph "恢复场景"
        R1["正常恢复:<br/>本地(1) + 云端(2) + 4S店(4)<br/>→ 3片足够"]
        R2["紧急恢复:<br/>云端(2) + 第三方(3) + 4S店(4)<br/>→ 需多方授权"]
    end

    S1 --> R1
    S2 --> R1
    S4 --> R1
    S2 --> R2
    S3 --> R2
    S4 --> R2

    style DRK fill:#ff8a80
    style SHAMIR fill:#fff9c4
```

### 5.4 TLS证书恢复流程

```mermaid
flowchart TD
    subgraph "触发条件"
        TRIGGER["• TEE证书读取失败<br/>• 证书完整性校验失败<br/>• 证书即将过期（提前30天预警）"]
    end

    TRIGGER --> STEP1

    subgraph "步骤1: 设备身份验证"
        STEP1["使用备用身份凭证"]
        STEP1_DETAIL["• 设备SN + MAC地址哈希<br/>• 带外通道联系车厂（4G/5G）<br/>• 车主身份确认（App授权/短信验证）"]
    end

    STEP1 --> STEP2

    subgraph "步骤2: 密钥恢复"
        STEP2["收集分片恢复密钥"]
        STEP2_DETAIL["• 收集分片: 本地 + 云端 + 4S店<br/>• TEE内执行Shamir恢复算法<br/>• 验证恢复的根密钥正确性"]
    end

    STEP2 --> STEP3

    subgraph "步骤3: 证书重签发"
        STEP3["申请新证书"]
        STEP3_DETAIL["• TEE内生成新密钥对<br/>• 生成P10证书请求（CSR）<br/>• 通过安全通道提交车厂CA<br/>• 车厂CA签发新证书"]
    end

    STEP3 --> STEP4

    subgraph "步骤4: 证书安装与验证"
        STEP4["安装新证书"]
        STEP4_DETAIL["• 将新证书安全写入TEE<br/>• 验证证书链完整性<br/>• 测试TLS连接<br/>• 更新证书版本号（防回退）"]
    end

    STEP4 --> STEP5

    subgraph "步骤5: 审计与通知"
        STEP5["完成恢复"]
        STEP5_DETAIL["• 记录证书恢复事件到安全日志<br/>• 上报车厂VSOC<br/>• 通知车主恢复完成"]
    end

    STEP1_DETAIL --> STEP1
    STEP2_DETAIL --> STEP2
    STEP3_DETAIL --> STEP3
    STEP4_DETAIL --> STEP4
    STEP5_DETAIL --> STEP5
```

---

## 六、Keystore功能集成方案

### 6.1 方案选择：直接使用Android Keystore

经过对比分析，**MTK平台建议直接使用Android Keystore API，无需额外集成SafeKey封装层**。理由如下：

#### 6.1.1 Keystore与SafeKey方案对比

| 对比维度 | 直接使用Android Keystore | 集成SafeKey封装 |
|---------|------------------------|----------------|
| **接口标准化** | ✅ 标准JCA/JCE接口，跨平台兼容 | ⚠️ 自定义接口，需要额外学习成本 |
| **硬件支持** | ✅ 直接调用Trustonic TEE的Keymaster TA | ⚠️ 通过SDF接口间接调用 |
| **密钥证明** | ✅ 原生支持Key Attestation | ❌ 需要额外开发 |
| **用户认证绑定** | ✅ 支持生物识别/PIN绑定 | ⚠️ 需要额外集成 |
| **系统维护成本** | ✅ Android系统自动更新 | ❌ 需要自行维护 |
| **开发复杂度** | ✅ 低，直接使用标准API | ⚠️ 高，需要维护中间层 |
| **安全审计** | ✅ 经过Google安全审计 | ⚠️ 需要自行审计 |
| **性能开销** | ✅ 直接调用，无额外开销 | ⚠️ 多一层封装，略有开销 |
| **故障恢复** | ✅ 系统级恢复机制 | ⚠️ 需要额外实现 |

#### 6.1.2 不集成SafeKey的理由

1. **功能重叠**：Android Keystore已提供完整的密钥管理能力，SafeKey的核心功能（密钥存储、密码运算）与Keystore高度重叠
2. **架构简化**：MTK平台仅有单一TEE安全锚点，减少中间层可降低系统复杂度和潜在故障点
3. **标准化优势**：使用标准API便于第三方应用集成和安全审计
4. **硬件直连**：Keystore直接调用Trustonic TEE的Keymaster TA，性能更优
5. **安全更新**：依赖Android系统安全更新，无需自行维护安全补丁
6. **单点风险**：MTK平台没有SE备份，增加中间层会增加额外的故障点

#### 6.1.3 推荐架构

```mermaid
graph TB
    subgraph "应用层"
        APP1["业务App"]
        APP2["车载服务"]
        APP3["第三方应用"]
    end

    subgraph "Android Framework"
        KS_API["Android Keystore API<br/>标准JCA/JCE接口"]
        KSP["KeyStore Provider"]
        KS2["Keystore2 Service"]
    end

    subgraph "HAL层"
        KM_HAL["KeyMint HAL"]
    end

    subgraph "安全硬件层"
        TEE["Trustonic TEE<br/>Kinibi<br/>Keymaster TA"]
    end

    APP1 --> KS_API
    APP2 --> KS_API
    APP3 --> KS_API
    KS_API --> KSP
    KSP --> KS2
    KS2 --> KM_HAL
    KM_HAL --> TEE

    style KS_API fill:#c8e6c9
    style TEE fill:#ffcdd2
```

#### 6.2.1 密钥生成示例

```java
// 使用Android Keystore生成硬件保护的密钥
KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_EC,
    "AndroidKeyStore"
);

KeyGenParameterSpec spec = new KeyGenParameterSpec.Builder(
    "vehicle_auth_key",
    KeyProperties.PURPOSE_SIGN | KeyProperties.PURPOSE_VERIFY
)
    .setDigests(KeyProperties.DIGEST_SHA256)
    .setUserAuthenticationRequired(true)
    .setUserAuthenticationParameters(0, KeyProperties.AUTH_BIOMETRIC_STRONG)
    .setAttestationChallenge(serverChallenge)  // 启用密钥证明
    .build();

keyPairGenerator.initialize(spec);
KeyPair keyPair = keyPairGenerator.generateKeyPair();

// 获取证明证书链，发送给服务器验证
Certificate[] certChain = keyStore.getCertificateChain("vehicle_auth_key");
```

#### 6.2.2 MTK平台特别注意事项

由于MTK平台没有独立的SE安全芯片，使用Keystore时需注意：

1. **无StrongBox支持**：不要调用`setIsStrongBoxBacked(true)`，会抛出异常
2. **TEE依赖**：所有硬件保护的密钥都依赖Trustonic TEE，需确保TEE健康
3. **故障降级**：为关键操作准备软件降级方案

```mermaid
flowchart TD
    START["需要存储密钥"] --> Q1{"安全要求?"}
    
    Q1 -->|高/最高| TEE["使用Trustonic TEE<br/>Keystore硬件保护"]
    
    Q1 -->|中| Q2{"需要硬件保护?"}
    Q2 -->|是| TEE
    Q2 -->|否| SW["软件Keystore<br/>不推荐用于敏感数据"]
    
    Q1 -->|低| SW
    
    TEE --> NOTE["注意：MTK无StrongBox<br/>不设置setIsStrongBoxBacked"]
    
    style TEE fill:#ffcdd2
    style SW fill:#fff9c4
    style NOTE fill:#e3f2fd
```

### 6.3 Keystore功能能力矩阵

| 功能类别 | 功能项 | 说明 | 业务场景 |
|---------|-------|------|---------|
| **密钥生成** | 对称密钥生成 | AES-128/256 | 本地数据加密 |
| | 非对称密钥生成 | RSA/EC | 签名验签、密钥交换 |
| | 硬件密钥 | TEE内生成 | 高安全等级场景 |
| **密钥存储** | 硬件安全存储 | 密钥不可导出 | 所有场景 |
| | 用户认证绑定 | 生物识别/PIN | 支付、敏感操作 |
| | 应用绑定 | 仅授权应用可用 | 隔离保护 |
| **密码操作** | 加密解密 | AES-GCM | 数据保护 |
| | 签名验签 | RSA/ECDSA | 身份认证、数据完整性 |
| | 密钥协商 | ECDH | 安全通信 |
| **密钥证明** | 硬件证明 | Key Attestation | 远程验证密钥安全性 |

### 6.4 业务使用场景

```mermaid
mindmap
  root((MTK平台Keystore业务场景))
    车载支付
      支付密钥保护
      交易签名
      生物认证绑定
    远程控制
      车辆解锁认证
      启动授权
      命令签名验证
    数据保护
      用户隐私数据加密
      行车记录保护
      设置信息加密
    安全通信
      TLS客户端认证
      端到端加密
      会话密钥协商
    OTA安全
      固件签名验证
      更新包加密
      回退保护
    DRM保护
      媒体内容解密
      许可证管理
      设备绑定
```

---

## 七、安全仲裁架构设计

### 7.1 操作安全级别分类

```mermaid
graph TB
    subgraph "安全级别分类"
        subgraph "CRITICAL 关键级别"
            C1["远程车辆控制（解锁、启动、窗户等）"]
            C2["OTA固件更新签名验证"]
            C3["设备根密钥操作"]
            C4["证书更新/恢复"]
            C_RULE["处理方式: 必须TEE执行，失败则拒绝操作"]
        end

        subgraph "HIGH 高级别"
            H1["TLS握手和通信加密"]
            H2["用户身份认证"]
            H3["支付/交易签名"]
            H4["隐私数据加密解密"]
            H_RULE["处理方式: 优先TEE，失败记录告警后可有限降级"]
        end

        subgraph "MEDIUM 中级别"
            M1["日志数据加密"]
            M2["配置数据保护"]
            M3["域间通信加密"]
            M4["缓存数据加密"]
            M_RULE["处理方式: 优先TEE，可降级到软件实现"]
        end

        subgraph "LOW 低级别"
            L1["临时会话密钥生成"]
            L2["随机数生成"]
            L3["哈希计算"]
            L4["非敏感数据加密"]
            L_RULE["处理方式: 可使用软件实现作为备选"]
        end
    end

    style C1 fill:#ff8a80
    style H1 fill:#ffcc80
    style M1 fill:#fff9c4
    style L1 fill:#c8e6c9
```

### 7.2 MTK平台安全仲裁决策流程

由于MTK平台没有SE备份，TEE故障时只能降级到软件实现或拒绝操作。

```mermaid
flowchart TD
    REQ["密码操作请求<br/>操作类型、调用者ID、上下文信息"]
    
    REQ --> STEP1["确定操作安全级别<br/>根据操作类型查询安全级别矩阵"]
    
    STEP1 --> STEP2["检查Trustonic TEE健康状态<br/>调用TEE健康检测模块"]
    
    STEP2 --> CHECK{"TEE状态?"}
    
    CHECK -->|正常| TEE_EXEC["TEE执行操作<br/>返回结果"]
    
    CHECK -->|异常/不可用| LEVEL_CHECK{"根据安全级别决策"}
    
    LEVEL_CHECK -->|CRITICAL| DENY1["拒绝操作<br/>记录告警<br/>无SE备份"]
    LEVEL_CHECK -->|HIGH| DENY2["拒绝操作<br/>启动恢复流程<br/>无SE备份"]
    LEVEL_CHECK -->|MEDIUM| DEGRADE1["降级到软件实现"]
    LEVEL_CHECK -->|LOW| DEGRADE2["使用软件实现"]
    
    DENY1 --> AUDIT["记录降级事件<br/>安全审计日志<br/>上报VSOC<br/>触发恢复流程"]
    DENY2 --> AUDIT
    DEGRADE1 --> AUDIT
    DEGRADE2 --> AUDIT

    style DENY1 fill:#ff8a80
    style DENY2 fill:#ff8a80
    style DEGRADE1 fill:#fff9c4
    style DEGRADE2 fill:#c8e6c9
```

### 7.3 软件密码备份模块

当TEE不可用时，为中低安全级别操作提供软件备选方案。

```mermaid
graph TB
    subgraph "MTK平台软件密码备份模块架构"
        subgraph "功能限制"
            ALLOW1["✓ 对称加密 AES-256-GCM"]
            ALLOW2["✓ 哈希计算 SHA-256, SM3"]
            ALLOW3["✓ 随机数生成 /dev/urandom"]
            DENY1["✗ 私钥签名（禁止 - 私钥必须在TEE）"]
            DENY2["✗ 设备认证（禁止 - 需要TEE密钥）"]
            DENY3["✗ 关键数据解密（禁止）"]
        end

        subgraph "安全措施"
            SEC1["密钥存储在内存中，进程退出即清除"]
            SEC2["使用白盒加密保护软件密钥"]
            SEC3["操作频率限制"]
            SEC4["所有操作记录审计日志"]
        end

        subgraph "使用场景"
            USE1["TEE暂时不可用时的日志加密"]
            USE2["非敏感配置数据的临时保护"]
            USE3["会话密钥的临时生成"]
        end

        subgraph "重要提醒"
            WARN["⚠ 软件实现安全性显著低于TEE<br/>仅作为临时备选方案<br/>TEE恢复后应立即切换回TEE执行<br/>MTK平台无SE备份，更需谨慎"]
        end
    end

    style DENY1 fill:#ff8a80
    style DENY2 fill:#ff8a80
    style DENY3 fill:#ff8a80
    style WARN fill:#fff9c4
```

---

## 八、故障检测与降级策略

### 8.1 TEE健康监测机制

```mermaid
graph TB
    subgraph "Trustonic TEE健康监测架构"
        subgraph "检测项目"
            CHK1["心跳检测<br/>周期: 5秒<br/>正常: 响应<100ms<br/>告警: 超时>1s"]
            CHK2["功能自检<br/>周期: 60秒<br/>正常: 成功<br/>告警: 失败"]
            CHK3["密钥完整性<br/>周期: 启动时+每小时<br/>正常: 校验通过<br/>告警: 校验失败"]
            CHK4["存储空间<br/>周期: 30分钟<br/>正常: >20%可用<br/>告警: <10%可用"]
            CHK5["操作延迟<br/>周期: 实时<br/>正常: P99<200ms<br/>告警: P99>500ms"]
            CHK6["错误率<br/>周期: 实时<br/>正常: <0.1%<br/>告警: >1%"]
        end

        subgraph "状态判定逻辑"
            HEALTHY["HEALTHY 正常<br/>所有检测项正常"]
            DEGRADED["DEGRADED 降级<br/>1-2项异常，核心功能可用"]
            FAILED["FAILED 失败<br/>心跳超时或核心功能失败"]
            RECOVERING["RECOVERING 恢复中<br/>正在执行恢复流程"]
        end
    end

    CHK1 --> HEALTHY
    CHK2 --> HEALTHY
    CHK3 --> HEALTHY
    CHK1 --> DEGRADED
    CHK2 --> DEGRADED
    CHK1 --> FAILED
    FAILED --> RECOVERING

    style HEALTHY fill:#c8e6c9
    style DEGRADED fill:#fff9c4
    style FAILED fill:#ff8a80
    style RECOVERING fill:#e3f2fd
```

### 8.2 MTK平台分级降级策略

由于MTK平台没有SE作为备份，降级策略更加保守。

```mermaid
stateDiagram-v2
    [*] --> FullSecurity: 系统启动

    FullSecurity: 正常模式 Full Security
    FullSecurity: • 所有密码操作在TEE执行
    FullSecurity: • 完整的远程控制功能
    FullSecurity: • 正常的TLS通信

    LimitedSecurity: 降级模式1 Limited Security
    LimitedSecurity: • 关键操作继续使用TEE（可能有延迟）
    LimitedSecurity: • 中低级别操作可使用软件备份
    LimitedSecurity: • 远程控制需要额外确认步骤
    LimitedSecurity: • 告警通知车厂VSOC
    LimitedSecurity: • 注意：无SE备份

    MinimalSecurity: 降级模式2 Minimal Security
    MinimalSecurity: • 关键操作全部拒绝（无SE备份）
    MinimalSecurity: • 远程控制禁用
    MinimalSecurity: • TLS通信使用缓存证书进行有限通信
    MinimalSecurity: • 本地功能部分可用（使用软件加密）
    MinimalSecurity: • 强制告警通知用户 + VSOC

    SafeMode: 安全模式 Safe Mode
    SafeMode: • 仅保留基本车辆控制（非智能功能）
    SafeMode: • 禁用所有网联功能
    SafeMode: • 强制显示警告信息
    SafeMode: • 建议用户前往4S店检修

    FullSecurity --> LimitedSecurity: TEE响应延迟升高/部分操作失败
    LimitedSecurity --> MinimalSecurity: TEE心跳超时/核心功能失败
    MinimalSecurity --> SafeMode: 持续故障>24小时/多次恢复失败

    LimitedSecurity --> FullSecurity: TEE恢复正常
    MinimalSecurity --> LimitedSecurity: TEE部分恢复
    SafeMode --> MinimalSecurity: 维修后恢复
```

---

## 九、安全监控与审计增强

### 9.1 安全日志架构

```mermaid
graph TB
    subgraph "MTK平台安全日志架构"
        subgraph "日志收集层"
            LOG1["TEE操作日志<br/>• 密码操作<br/>• 密钥访问<br/>• 认证事件"]
            LOG2["仲裁决策日志<br/>• 决策原因<br/>• 安全级别<br/>• 执行模块"]
            LOG3["降级事件日志<br/>• 模式变更<br/>• 功能状态<br/>• 恢复尝试"]
        end

        subgraph "安全日志聚合器"
            AGG["SecurityLogAggregator"]
            AGG_SEC["安全措施:<br/>• 日志签名: TEE密钥HMAC签名<br/>• 日志加密: 敏感内容加密存储<br/>• 防篡改: 日志链式哈希<br/>• 循环存储: 固定大小防耗尽<br/>• 安全时间戳: 可信时间源"]
        end

        subgraph "存储与上报"
            LOCAL["本地安全存储<br/>RPMB/加密分区<br/>• 保留最近30天日志<br/>• 支持离线审计"]
            CLOUD["云端VSOC上报<br/>加密通道<br/>• 实时上报严重事件<br/>• 定期批量上报"]
        end
    end

    LOG1 --> AGG
    LOG2 --> AGG
    LOG3 --> AGG
    AGG --> AGG_SEC
    AGG_SEC --> LOCAL
    AGG_SEC --> CLOUD
```

### 9.2 关键安全监控指标

```mermaid
graph TB
    subgraph "关键安全监控指标"
        subgraph "TEE健康指标"
            TEE1["TEE心跳延迟 >500ms → 记录日志"]
            TEE2["TEE心跳超时 >3次/分钟 → 触发降级"]
            TEE3["TEE操作失败率 >1% → 告警+检查"]
            TEE4["TEE存储使用 >90% → 告警+清理"]
        end

        subgraph "密钥安全指标"
            KEY1["密钥访问失败 任何 → 记录+检查"]
            KEY2["密钥校验失败 任何 → 告警+恢复"]
            KEY3["异常密钥操作 模式异常 → 告警+审查"]
        end

        subgraph "认证安全指标"
            AUTH1["认证失败次数 >5次/分钟 → 临时锁定"]
            AUTH2["暴力破解检测 触发阈值 → 锁定+告警"]
            AUTH3["认证来源异常 非法调用 → 拒绝+告警"]
        end

        subgraph "系统状态指标"
            SYS1["降级事件 任何 → 记录+上报"]
            SYS2["恢复失败 >3次 → 进入安全模式"]
            SYS3["模式变更 任何 → 通知+上报"]
        end

        subgraph "通信安全指标"
            COMM1["TLS握手失败 >5% → 告警+检查证书"]
            COMM2["证书即将过期 <30天 → 告警+准备更新"]
            COMM3["证书验证失败 任何 → 拒绝连接+告警"]
        end
    end
```

---

## 十、实施建议

### 10.1 MTK平台实施优先级排序

```mermaid
gantt
    title MTK平台安全增强实施优先级
    dateFormat  YYYY-MM-DD
    section P0 立即实施
    TEE健康监测机制           :p0_1, 2025-01-01, 30d
    安全审计日志增强           :p0_2, after p0_1, 20d
    基础仲裁逻辑               :p0_3, after p0_1, 25d
    section P1 短期实施
    降级策略管理               :p1_1, after p0_3, 30d
    密钥派生架构               :p1_2, after p0_2, 35d
    VSOC对接                   :p1_3, after p1_1, 25d
    section P2 中期实施
    密钥分片备份               :p2_1, after p1_2, 40d
    密钥恢复流程               :p2_2, after p2_1, 35d
    软件密码备份模块           :p2_3, after p1_1, 30d
    Keystore功能集成           :p2_4, after p1_2, 40d
    section P3 长期优化
    证书自动更新机制           :p3_1, after p2_2, 30d
    异常行为检测增强           :p3_2, after p2_3, 35d
    安全合规审计工具           :p3_3, after p3_1, 30d
```

### 10.2 MTK平台特别注意事项

```mermaid
mindmap
  root((MTK平台<br/>风险与限制))
    架构限制
      仅TEE单安全锚点
      无SE硬件备份
      物理攻击防护能力有限
      软件备份方案安全性低
    实施风险
      TEE故障无硬件备份
      降级策略可能影响用户体验
      密钥恢复流程需要车厂支持
    缓解措施
      更频繁的TEE健康监测
      完善的密钥分片备份
      严格限制软件备份功能
      完善的监控告警机制
    未来演进建议
      后续车型考虑增加SE
      评估新一代安全架构
      持续关注TEE安全漏洞
```

### 10.3 与现有系统集成方案

```mermaid
graph TB
    subgraph "MTK平台与现有系统集成方案"
        subgraph "现有架构"
            SKM_OLD["SafeKeyManager (现有)"]
            SKS_OLD["SafeKeyService (现有)"]
            MW_OLD["中间件层 (现有)"]
            TEE_OLD["Trustonic TEE (现有)"]
        end

        subgraph "新增组件"
            ARB["SecurityArbiterWrapper<br/>安全仲裁封装层"]
            ARB_DESC["• 拦截所有密码操作请求<br/>• 执行安全级别判定<br/>• 调用仲裁引擎决策<br/>• 记录审计日志"]
            
            SW_CRYPTO["SoftwareCryptoFallback<br/>软件密码备份"]
            
            KS_INT["Keystore集成模块"]
        end

        subgraph "集成原则"
            RULE1["对上层API保持兼容"]
            RULE2["安全仲裁层对调用者透明"]
            RULE3["降级和恢复自动进行"]
            RULE4["安全事件通过统一通道上报"]
        end
    end

    SKM_OLD --> ARB
    ARB --> SKS_OLD
    ARB --> KS_INT
    SKS_OLD --> MW_OLD
    MW_OLD --> TEE_OLD
    MW_OLD --> SW_CRYPTO

    style ARB fill:#c8e6c9
    style SW_CRYPTO fill:#c8e6c9
    style KS_INT fill:#c8e6c9
```

---

## 参考资料

### 标准与法规
- ISO/SAE 21434:2021 - 道路车辆网络安全工程
- UNECE WP.29 R155 - 网络安全管理系统
- GM/T 0018 - 密码设备应用接口规范

### 技术参考
- ARM TrustZone技术白皮书
- Trustonic Kinibi TEE文档
- Android Keystore系统架构
- Shamir Secret Sharing算法
- HKDF密钥派生函数 (RFC 5869)

---

*文档版本: 2.0*  
*更新日期: 2025-01-20*  
*适用架构: MTK平台(Trustonic TEE)*
