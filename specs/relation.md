# Android 安全架构核心概念与关系

## 一、概念介绍

### 1. TEE (Trusted Execution Environment)

**可信执行环境**，是与主操作系统隔离的安全执行环境。

- **硬件基础**：通常基于 ARM TrustZone 技术实现
- **组成**：包含独立的 TEE OS（如 Trusty、QSEE、iTrustee）
- **特点**：
  - 与 Android（Rich OS）并行运行
  - 拥有独立的安全存储区域
  - 硬件级别的内存隔离
- **作用**：为敏感操作提供隔离的执行环境

```
┌─────────────────────────────────────────────────────┐
│                    硬件层 (SoC)                      │
├────────────────────────┬────────────────────────────┤
│     Normal World       │      Secure World          │
│  ┌──────────────────┐  │  ┌──────────────────────┐  │
│  │   Android OS     │  │  │      TEE OS          │  │
│  │  (Rich OS)       │  │  │  (Trusty/QSEE等)     │  │
│  │                  │  │  │                      │  │
│  │  - Apps          │  │  │  - Keymaster TA      │  │
│  │  - Framework     │  │  │  - Gatekeeper TA     │  │
│  │  - HAL           │  │  │  - Fingerprint TA    │  │
│  └──────────────────┘  │  └──────────────────────┘  │
└────────────────────────┴────────────────────────────┘
```

---

### 2. Keystore / Keystore2

**Android 密钥存储系统**，为应用提供密钥管理和加密操作的统一API。

#### Keystore (传统版本)
- Android 4.3 引入
- 运行在 `system_server` 进程中
- 通过 Binder 与应用通信

#### Keystore2 (Android 12+)
- 完全重写，使用 Rust 实现
- 独立进程运行（`/system/bin/keystore2`）
- 更好的安全性和性能
- 支持 AIDL 接口

**主要功能**：
- 密钥的生成、导入、删除
- 密钥的访问控制
- 与底层 Keymaster/KeyMint 通信
- 密钥使用授权管理

---

### 3. Keymaster / KeyMint

**密钥管理的硬件抽象层（HAL）**，定义了密钥操作的标准接口。

#### Keymaster (旧版)
- Keymaster 1.0 - 4.1 版本
- 使用 HIDL 接口
- 在 TEE 中运行的 TA（Trusted Application）实现

#### KeyMint (Android 12+)
- 替代 Keymaster 的新 HAL
- 使用 AIDL 接口
- 支持更多算法和特性
- 版本：KeyMint 1.0, 2.0, 3.0

**核心职责**：
```
┌─────────────────────────────────────────┐
│            Keymaster/KeyMint            │
├─────────────────────────────────────────┤
│  • 密钥生成（RSA, EC, AES, HMAC等）     │
│  • 密钥导入/导出                        │
│  • 加密/解密操作                        │
│  • 签名/验签操作                        │
│  • 密钥绑定（硬件、用户认证等）         │
│  • Key Attestation（密钥证明）          │
└─────────────────────────────────────────┘
```

---

### 4. StrongBox

**基于独立安全芯片（Secure Element）的密钥存储**。

- **硬件要求**：独立的安全处理器（非共享CPU）
- **Android 9 (API 28)** 引入
- **安全级别**：高于 TEE

**StrongBox vs TEE**：

| 特性 | TEE | StrongBox |
|------|-----|-----------|
| 处理器 | 共享主CPU | 独立安全芯片 |
| 物理攻击抵抗 | 中等 | 强 |
| 性能 | 较快 | 较慢 |
| 算法支持 | 全面 | 有限（RSA 2048, EC P-256, AES-128/256） |

**使用方式**：
```java
KeyGenParameterSpec spec = new KeyGenParameterSpec.Builder(alias, purposes)
    .setIsStrongBoxBacked(true)  // 指定使用StrongBox
    .build();
```

---

### 5. GateKeeper

**锁屏凭证验证服务**，负责处理 PIN、密码、图案的验证。

**架构**：
```
┌──────────────────┐
│  LockSettingsService (Java)       │
└─────────┬────────┘
          │ Binder
┌─────────▼────────┐
│  gatekeeperd (Native Daemon)      │
└─────────┬────────┘
          │ HAL
┌─────────▼────────┐
│  Gatekeeper HAL (HIDL/AIDL)       │
└─────────┬────────┘
          │
┌─────────▼────────┐
│  Gatekeeper TA (TEE)              │
└──────────────────┘
```

**核心功能**：
- 存储密码哈希（在TEE安全存储中）
- 验证用户输入的凭证
- 生成 **AuthToken**（认证令牌）
- 防暴力破解（指数级延迟）

**AuthToken 结构**：
```c
struct HardwareAuthToken {
    uint64_t challenge;
    uint64_t userId;
    uint64_t authenticatorId;
    uint32_t authenticatorType;  // PASSWORD = 1, FINGERPRINT = 2
    uint64_t timestamp;
    uint8_t hmacKey[32];         // 由TEE中的共享密钥签名
};
```

---

### 6. 指纹认证 (Fingerprint)

**指纹识别系统**，提供生物特征认证能力。

**架构层次**：
```
┌─────────────────────────────────────┐
│  FingerprintManager / BiometricPrompt (API)  │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│  FingerprintService (System Service)         │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│  fingerprintd (Native Daemon)                │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│  Fingerprint HAL (HIDL/AIDL)                 │
└─────────────────┬───────────────────┘
                  │
┌────────────┬────▼────┬──────────────┐
│ 指纹传感器  │  TEE TA  │  安全存储    │
└────────────┴─────────┴──────────────┘
```

**安全特性**：
- 指纹模板**只存储在TEE**中，不会进入Android系统
- 匹配操作在TEE中完成
- 成功认证后生成 AuthToken

---

### 7. 生物认证 (BiometricPrompt)

**统一的生物认证框架**（Android 9+），整合多种生物识别方式。

**支持的认证方式**：
- 指纹 (Fingerprint)
- 面部识别 (Face)
- 虹膜识别 (Iris)

**安全等级分类**：
```
┌─────────────────────────────────────────────┐
│  BIOMETRIC_STRONG (Class 3)                 │
│  - 欺骗接受率 (SAR) < 7%                    │
│  - 可用于 Keystore 密钥授权                 │
├─────────────────────────────────────────────┤
│  BIOMETRIC_WEAK (Class 2)                   │
│  - SAR < 20%                                │
│  - 仅用于解锁，不能用于密钥授权             │
├─────────────────────────────────────────────┤
│  BIOMETRIC_CONVENIENCE (Class 1)            │
│  - 便捷性优先                               │
│  - 安全性最低                               │
└─────────────────────────────────────────────┘
```

---

### 8. Key Attestation（密钥证明）

**证明密钥确实由安全硬件生成和保护的机制**。

**工作原理**：
```
┌──────────┐    1. 请求生成密钥+Attestation    ┌──────────┐
│   App    │ ─────────────────────────────────▶│ Keystore │
└──────────┘                                   └────┬─────┘
     ▲                                              │
     │                                              ▼
     │         2. 在TEE中生成密钥              ┌──────────┐
     │            并用设备密钥签名证书链        │ TEE/SE   │
     │                                         └────┬─────┘
     │                                              │
     │    3. 返回证书链                             │
     └──────────────────────────────────────────────┘

     4. App 将证书链发送给服务器验证
        ┌──────────┐
        │  Server  │  验证：
        └──────────┘  - 证书链是否由Google根证书签发
                      - 密钥属性是否符合要求
                      - 设备是否可信
```

**证书链结构**：
```
┌─────────────────────────────────────┐
│  Root Certificate (Google)          │  ← 预置在服务端
├─────────────────────────────────────┤
│  Intermediate Certificate           │  ← 设备厂商证书
├─────────────────────────────────────┤
│  Attestation Certificate            │  ← 包含密钥属性的证书
│  (包含KeyDescription扩展)           │
└─────────────────────────────────────┘
```

**KeyDescription 包含的信息**：
- 密钥用途 (purpose)
- 算法和参数
- 安全级别 (Software/TEE/StrongBox)
- 是否需要用户认证
- Boot 状态信息
- 设备ID信息（可选）

---

## 二、架构关系图

### 整体架构

```
┌────────────────────────────────────────────────────────────────────┐
│                         Android 应用层                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ KeyStore API │  │BiometricPrompt│  │   其他安全相关 API       │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────────────┘  │
└─────────┼─────────────────┼────────────────────────────────────────┘
          │                 │
┌─────────▼─────────────────▼────────────────────────────────────────┐
│                      Android Framework                              │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────┐  │
│  │ KeyStore       │  │ BiometricService│  │ LockSettingsService │  │
│  │ Service        │  │                │  │                      │  │
│  └───────┬────────┘  └───────┬────────┘  └──────────┬───────────┘  │
└──────────┼───────────────────┼──────────────────────┼──────────────┘
           │                   │                      │
┌──────────▼───────────────────▼──────────────────────▼──────────────┐
│                        Native Services                              │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────┐  │
│  │   keystore2    │  │  fingerprintd  │  │    gatekeeperd       │  │
│  │   (Rust)       │  │                │  │                      │  │
│  └───────┬────────┘  └───────┬────────┘  └──────────┬───────────┘  │
└──────────┼───────────────────┼──────────────────────┼──────────────┘
           │                   │                      │
┌──────────▼───────────────────▼──────────────────────▼──────────────┐
│                     HAL (Hardware Abstraction Layer)                │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────┐  │
│  │ KeyMint HAL    │  │ Fingerprint HAL│  │   Gatekeeper HAL     │  │
│  │ (AIDL)         │  │ (AIDL)         │  │   (AIDL)             │  │
│  └───────┬────────┘  └───────┬────────┘  └──────────┬───────────┘  │
└──────────┼───────────────────┼──────────────────────┼──────────────┘
           │                   │                      │
           ▼                   ▼                      ▼
┌────────────────────────────────────────────────────────────────────┐
│                        TEE (Secure World)                          │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────┐  │
│  │ KeyMint TA     │  │ Fingerprint TA │  │   Gatekeeper TA      │  │
│  │                │  │                │  │                      │  │
│  │ • 密钥生成     │  │ • 模板存储     │  │ • 密码哈希存储       │  │
│  │ • 加密操作     │  │ • 指纹匹配     │  │ • 凭证验证           │  │
│  │ • Attestation  │  │ • AuthToken    │  │ • AuthToken生成      │  │
│  └────────────────┘  └────────────────┘  └──────────────────────┘  │
│                                                                     │
│                    ┌──────────────────────┐                        │
│                    │   共享 AuthToken Key │  (HMAC密钥)            │
│                    └──────────────────────┘                        │
└────────────────────────────────────────────────────────────────────┘
           │
           ▼ (可选)
┌────────────────────────────────────────────────────────────────────┐
│                    StrongBox (独立安全芯片)                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  独立 CPU │ 独立 RAM │ 安全存储 │ KeyMint 实现              │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

### AuthToken 流转机制

```
用户输入密码/指纹
        │
        ▼
┌───────────────────┐     验证请求      ┌───────────────────┐
│  GateKeeper/      │ ────────────────▶ │      TEE          │
│  Fingerprint      │                   │                   │
│  Service          │ ◀──────────────── │  验证通过后       │
└───────────────────┘    AuthToken      │  生成AuthToken    │
        │                (HMAC签名)      └───────────────────┘
        │
        ▼ 存储AuthToken
┌───────────────────┐
│   Keystore2       │
└───────────────────┘
        │
        │ 应用请求使用密钥
        ▼
┌───────────────────┐     密钥操作请求   ┌───────────────────┐
│   KeyMint HAL     │ ────────────────▶ │      TEE          │
│                   │   + AuthToken     │                   │
│                   │                   │  验证AuthToken    │
│                   │ ◀──────────────── │  执行密钥操作     │
└───────────────────┘      操作结果      └───────────────────┘
```

---

## 三、实际业务场景

### 场景1：银行App登录

**需求**：用户使用指纹快速登录银行App

**流程**：
```
1. 首次设置（绑定指纹）
   ┌──────────────────────────────────────────────────────────┐
   │ App 生成 RSA 密钥对，要求：                              │
   │ - setUserAuthenticationRequired(true)                    │
   │ - setUserAuthenticationValidityDurationSeconds(-1)       │
   │   (每次使用都需要认证)                                   │
   │ - setInvalidatedByBiometricEnrollment(true)              │
   │   (新增指纹时密钥失效)                                   │
   └──────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌──────────────────────────────────────────────────────────┐
   │ 将公钥上传到服务器，与用户账号绑定                        │
   └──────────────────────────────────────────────────────────┘

2. 指纹登录
   ┌──────────────────────────────────────────────────────────┐
   │ 服务器下发随机 challenge                                 │
   └──────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌──────────────────────────────────────────────────────────┐
   │ App 调用 BiometricPrompt 进行指纹认证                    │
   │ → 用户按指纹                                             │
   │ → TEE 验证指纹，生成 AuthToken                           │
   └──────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌──────────────────────────────────────────────────────────┐
   │ App 使用私钥签名 challenge                               │
   │ → Keystore 验证 AuthToken 有效                           │
   │ → TEE 中执行签名操作                                     │
   └──────────────────────────────────────────────────────────┘
                              │
                              ▼
   ┌──────────────────────────────────────────────────────────┐
   │ 将签名发送给服务器验证                                   │
   │ → 服务器用公钥验证签名                                   │
   │ → 验证通过，登录成功                                     │
   └──────────────────────────────────────────────────────────┘
```

**代码示例**：
```kotlin
// 生成密钥
val keyGenSpec = KeyGenParameterSpec.Builder(
    "bank_login_key",
    KeyProperties.PURPOSE_SIGN
)
    .setDigests(KeyProperties.DIGEST_SHA256)
    .setSignaturePaddings(KeyProperties.SIGNATURE_PADDING_RSA_PKCS1)
    .setUserAuthenticationRequired(true)
    .setUserAuthenticationParameters(
        0,  // 每次都需要认证
        KeyProperties.AUTH_BIOMETRIC_STRONG
    )
    .setInvalidatedByBiometricEnrollment(true)
    .build()

val keyPairGenerator = KeyPairGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_RSA, "AndroidKeyStore"
)
keyPairGenerator.initialize(keyGenSpec)
keyPairGenerator.generateKeyPair()

// 使用指纹认证后签名
val biometricPrompt = BiometricPrompt(activity, executor, callback)
val signature = Signature.getInstance("SHA256withRSA")
signature.initSign(privateKey)

val cryptoObject = BiometricPrompt.CryptoObject(signature)
biometricPrompt.authenticate(promptInfo, cryptoObject)

// 在回调中完成签名
override fun onAuthenticationSucceeded(result: AuthenticationResult) {
    val signature = result.cryptoObject?.signature
    signature?.update(challenge)
    val signedData = signature?.sign()
    // 发送给服务器验证
}
```

---

### 场景2：设备完整性验证（Key Attestation）

**需求**：服务器验证客户端是否为真实、未被篡改的Android设备

**流程**：
```
┌──────────────────────────────────────────────────────────────┐
│  1. 服务器下发 challenge (随机数)                            │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  2. App 生成密钥并请求 Attestation                           │
│                                                              │
│  KeyGenParameterSpec.Builder(alias, PURPOSE_SIGN)            │
│      .setAttestationChallenge(challenge)  // 关键参数        │
│      .build()                                                │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  3. 获取证书链                                               │
│                                                              │
│  val certChain = keyStore.getCertificateChain(alias)         │
│  // certChain[0] = Attestation Certificate (包含密钥属性)    │
│  // certChain[1] = Intermediate Certificate                  │
│  // certChain[2] = Root Certificate                          │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  4. 服务器验证证书链                                         │
│                                                              │
│  a. 验证证书链签名（根证书是Google预置的）                   │
│  b. 解析 Attestation Extension (OID: 1.3.6.1.4.1.11129.2.1.17)│
│  c. 检查 challenge 是否匹配                                  │
│  d. 检查安全级别 (TEE/StrongBox)                             │
│  e. 检查 Boot 状态 (Verified Boot)                           │
│  f. 检查设备是否在 CRL 吊销列表中                            │
└──────────────────────────────────────────────────────────────┘
```

**Attestation Extension 关键字段**：
```
KeyDescription ::= SEQUENCE {
    attestationVersion  INTEGER,
    attestationSecurityLevel  SecurityLevel,  -- Software(0), TEE(1), StrongBox(2)
    keymasterVersion  INTEGER,
    keymasterSecurityLevel  SecurityLevel,
    attestationChallenge  OCTET_STRING,       -- 服务器下发的challenge
    uniqueId  OCTET_STRING,
    softwareEnforced  AuthorizationList,
    teeEnforced  AuthorizationList,           -- TEE强制执行的属性
}

RootOfTrust ::= SEQUENCE {
    verifiedBootKey  OCTET_STRING,
    deviceLocked  BOOLEAN,
    verifiedBootState  VerifiedBootState,     -- Verified, SelfSigned, Unverified, Failed
    verifiedBootHash  OCTET_STRING,
}
```

---

### 场景3：支付App交易签名

**需求**：使用StrongBox保护支付密钥，每笔交易需要用户认证

**流程**：
```
┌──────────────────────────────────────────────────────────────┐
│  密钥生成（注册时）                                          │
│                                                              │
│  KeyGenParameterSpec.Builder("payment_key", PURPOSE_SIGN)    │
│      .setIsStrongBoxBacked(true)        // 使用StrongBox     │
│      .setUserAuthenticationRequired(true)                    │
│      .setUserAuthenticationParameters(                       │
│          0,                                                  │
│          AUTH_BIOMETRIC_STRONG or AUTH_DEVICE_CREDENTIAL     │
│      )                                                       │
│      .setAttestationChallenge(serverChallenge)               │
│      .build()                                                │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  服务器验证 Attestation                                      │
│                                                              │
│  - 确认 attestationSecurityLevel == StrongBox               │
│  - 确认 userAuthenticationRequired == true                   │
│  - 确认设备处于 Verified Boot 状态                           │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  交易签名流程                                                │
│                                                              │
│  1. 用户发起支付，输入金额                                   │
│  2. App 显示 BiometricPrompt                                │
│  3. 用户验证指纹/面部/PIN                                    │
│  4. 认证成功后，在 StrongBox 中签名交易数据                  │
│  5. 将签名发送给服务器                                       │
└──────────────────────────────────────────────────────────────┘
```

---

### 场景4：文件加密存储

**需求**：加密存储敏感文件，需要用户认证后才能解密

```kotlin
// 生成AES密钥
val keyGenSpec = KeyGenParameterSpec.Builder(
    "file_encryption_key",
    KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
)
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setUserAuthenticationRequired(true)
    .setUserAuthenticationParameters(
        300,  // 认证后5分钟内有效
        KeyProperties.AUTH_BIOMETRIC_STRONG or
        KeyProperties.AUTH_DEVICE_CREDENTIAL
    )
    .build()

// 加密文件
fun encryptFile(plainData: ByteArray): ByteArray {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.ENCRYPT_MODE, secretKey)
    val iv = cipher.iv
    val encrypted = cipher.doFinal(plainData)
    return iv + encrypted  // IV + 密文
}

// 解密文件（需要先通过BiometricPrompt认证）
fun decryptFile(encryptedData: ByteArray): ByteArray {
    val iv = encryptedData.sliceArray(0..11)
    val cipherText = encryptedData.sliceArray(12 until encryptedData.size)

    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.DECRYPT_MODE, secretKey, GCMParameterSpec(128, iv))
    return cipher.doFinal(cipherText)
}
```

---

## 四、版本演进

| Android 版本 | 安全特性 |
|-------------|---------|
| 4.3 (API 18) | Keystore 系统引入 |
| 6.0 (API 23) | 指纹 API、Keymaster 1.0 |
| 7.0 (API 24) | Key Attestation、Keymaster 2.0 |
| 8.0 (API 26) | ID Attestation、Keymaster 3.0 |
| 9.0 (API 28) | StrongBox、BiometricPrompt、Keymaster 4.0 |
| 10 (API 29) | 生物识别安全等级分类 |
| 11 (API 30) | 身份凭证 API (Identity Credential) |
| 12 (API 31) | Keystore2 (Rust)、KeyMint 1.0 |
| 13 (API 33) | KeyMint 2.0 |
| 14 (API 34) | KeyMint 3.0、更严格的 Attestation |

---

## 五、安全建议

### 密钥管理最佳实践

1. **优先使用硬件安全**
   ```kotlin
   // 检查StrongBox可用性
   if (packageManager.hasSystemFeature(PackageManager.FEATURE_STRONGBOX_KEYSTORE)) {
       spec.setIsStrongBoxBacked(true)
   }
   ```

2. **合理设置用户认证**
   ```kotlin
   // 敏感操作：每次都需要认证
   setUserAuthenticationParameters(0, AUTH_BIOMETRIC_STRONG)

   // 一般操作：认证后一段时间内有效
   setUserAuthenticationParameters(300, AUTH_BIOMETRIC_STRONG or AUTH_DEVICE_CREDENTIAL)
   ```

3. **使用 Key Attestation 验证设备**
   - 服务端必须验证证书链
   - 检查设备是否在吊销列表中
   - 验证 Boot 状态

4. **处理密钥失效**
   ```kotlin
   try {
       cipher.init(Cipher.DECRYPT_MODE, key)
   } catch (e: KeyPermanentlyInvalidatedException) {
       // 密钥已失效（如新增了指纹），需要重新注册
   } catch (e: UserNotAuthenticatedException) {
       // 需要用户认证
   }
   ```

---

## 六、总结

| 组件 | 层级 | 职责 |
|-----|------|------|
| BiometricPrompt | API | 统一生物认证接口 |
| Keystore API | API | 密钥管理接口 |
| Keystore2 | System Service | 密钥存储服务 |
| BiometricService | System Service | 生物认证服务 |
| LockSettingsService | System Service | 锁屏设置服务 |
| KeyMint HAL | HAL | 密钥操作硬件抽象 |
| Fingerprint HAL | HAL | 指纹硬件抽象 |
| Gatekeeper HAL | HAL | 凭证验证硬件抽象 |
| TEE | 安全环境 | 可信执行环境 |
| StrongBox | 安全硬件 | 独立安全芯片 |
| Key Attestation | 安全机制 | 密钥/设备证明 |
