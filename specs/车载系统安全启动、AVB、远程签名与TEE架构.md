# **深度研究报告：车载系统安全启动、AVB、远程签名与TEE架构**

## **1. 执行摘要与行业背景**

随着汽车工业向"软件定义汽车"（Software Defined Vehicle,
SDV）架构的转型，车辆电子电气架构（E/E架构）正经历从分布式电子控制单元（ECU）向域控制器（Domain
Controller）及中央计算平台（Zonal/Central
Compute）的剧烈演变。在这一进程中，车载系统的代码量已呈指数级增长，高端车型集成的代码行数往往超过1亿行
。这种复杂性急剧扩大了攻击面，使得确保软件执行环境的完整性与可信性成为车辆功能安全（Safety）与网络安全（Security）的基石。

本报告旨在对车载系统的核心安全机制------安全启动（Secure
Boot）、Android验证启动（Android Verified Boot,
AVB）、远程签名基础设施（Remote Signing
Infrastructure）以及可信执行环境（TEE）------进行详尽的技术剖析。分析不仅涵盖各技术的底层实现原理与架构依赖，还将深入探讨其在虚拟化环境下的交互逻辑、符合ISO/SAE
21434与UN R155法规的合规性设计，以及面向后量子密码学（Post-Quantum
Cryptography, PQC）的未来演进路径。

报告指出，现代车载安全架构已不再是单一技术的堆叠，而是一个以硬件信任根（Root
of Trust,
RoT）为锚点，通过加密链条贯穿引导加载程序（Bootloader）、管理程序（Hypervisor）、操作系统内核及用户空间的纵深防御体系。特别是随着云端CI/CD流水线与车载OTA（Over-the-Air）更新的紧密耦合，远程签名机制与密钥管理策略已成为供应链安全的关键环节。

## **2. 硬件信任根（Hardware Root of Trust）与基础架构**

安全启动的本质在于建立一条信任链（Chain of Trust,
CoT），而这条链的起点必须是一个不可篡改的信任根。在车载SoC（System on
Chip）中，信任根通常由片上引导只读存储器（Boot ROM/Mask
ROM）与一次性可编程存储器（One-Time Programmable, OTP/eFuses）共同构成
。

### **2.1 掩膜ROM（Mask ROM）与引导流程起点**

Mask
ROM是SoC在制造阶段光刻在硅片上的固化代码，具有物理上的不可篡改性。它是系统上电复位后执行的第一段指令。Mask
ROM的主要职责是初始化最基本的系统资源（如内部SRAM），并从外部存储介质（如eMMC、UFS、SPI
NOR Flash）加载第一阶段引导加载程序（Primary Boot Loader, PBL）。

在执行PBL之前，Mask ROM会读取存储在OTP/eFuses中的信任根公钥哈希（Root of
Trust Public Key Hash, ROTPK
Hash），并计算PBL头部携带的公钥的哈希值进行比对。如果匹配，再使用该公钥验证PBL固件的数字签名。这一过程确保了即便是最底层的引导代码也必须经过OEM或芯片厂商的授权
。

### **2.2 电子熔丝（eFuses）与生命周期管理**

OTP存储器（通常实现为eFuses）在车辆安全生命周期管理中扮演着至关重要的角色。除了存储ROTPK哈希外，eFuses还用于定义设备的安全状态和硬件配置：

-   **安全启动使能（Secure Boot Enable）：**
    > 一个特定的熔丝位，一旦熔断，Mask
    > ROM将强制执行签名验证。如果验证失败，系统将拒绝启动或进入受限的恢复模式
    > 。

-   **防回滚计数器（Anti-Rollback Counter）：** 为了防止重放攻击（Replay
    > Attack）或降级攻击，eFuses中预留了多组单调计数器。每次固件更新时，若新固件的安全版本号（Security
    > Version Number,
    > SVN）高于当前熔丝值，系统将熔断更多的熔丝位以更新计数器。由于熔丝不可恢复，攻击者无法将系统回滚到存在已知漏洞的旧版本固件
    > 。

-   **调试端口禁用（JTAG Disable）：**
    > 在生产阶段结束后，通过熔断特定熔丝永久关闭JTAG/SWD等硬件调试接口，防止物理攻击者通过调试器读取内存或篡改寄存器
    > 。

### **2.3 物理不可克隆函数（PUF）**

随着半导体工艺的进步，越来越多的车规级芯片开始集成物理不可克隆函数（PUF）。PUF利用芯片制造过程中的微小物理差异（如SRAM上电初始状态的随机性）来生成唯一的数字指纹
。

在车载密钥管理中，PUF的主要优势在于其输出的密钥无需存储在非易失性存储器（NVM）中。当芯片断电时，密钥即消失；上电时，PUF电路重新生成相同的密钥。这种机制极大地增强了对抗物理侵入式攻击（如解剖芯片探测存储单元）的能力。PUF生成的密钥通常用作密钥加密密钥（Key
Encryption Key,
KEK），用于加密存储在Flash中的实际业务密钥（如用于AVB验证的公钥或TEE的存储密钥）。

### **2.4 安全组件对比：HSM vs. SHE vs. TEE vs. TPM**

在车载电子架构设计中，混淆HSM、SHE和TEE的概念是常见的误区。这三者虽然都服务于安全目的，但在架构层次、隔离级别和功能定位上存在显著差异。

  ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  **特性**            **SHE (Secure Hardware Extension)**             **HSM (Hardware Security Module)**                              **TEE (Trusted Execution Environment)**                               **TPM (Trusted Platform Module)**
  ------------------- ----------------------------------------------- --------------------------------------------------------------- --------------------------------------------------------------------- --------------------------------------------------------
  **定义**            AUTOSAR定义的标准功能规范，通常作为外设存在。   独立的物理核心，拥有专属内存和总线，物理隔离。                  主处理器（如ARM Cortex-A）的安全执行模式（TrustZone），逻辑隔离。     独立的安全芯片，通常通过SPI/LPC总线连接。

  **隔离性**          逻辑/物理隔离取决于实现，通常共享主核资源。     **物理隔离**：拥有独立的CPU核、RAM和Flash，主核无法直接访问。   **逻辑隔离**：通过硬件防火墙（TZASC）隔离内存，时间片共享主核算力。   **物理隔离**：完全独立的芯片封装，抗物理攻击能力最强。

  **性能**            低：通常仅提供AES硬件加速和基本的密钥存储。     中/高：可编程，支持非对称加密（RSA/ECC）加速，适合高频交互。    **极高**：利用主核GHz级算力，适合复杂算法和生物识别。                 低：受限于总线速度（SPI），主要用于密钥存储和PCR度量。

  **用途**            基本的密钥存储、AES加密、MAC验证。              车身控制、网关安全、SecOC密钥管理、主密钥存储 。                DRM、Android KeyMint实现、受信任的用户交互（TUI）、复杂业务逻辑 。    远程证明（Attestation）、PC级架构的信任根。

  **主要厂商/实现**   NXP CSE (Cryptographic Services Engine)         Infineon AURIX HSM, Renesas RH850 HSM, Vector vHSM              Qualcomm QTEE, OP-TEE, Trusty                                         Infineon OPTIGA TPM, STMicroelectronics ST33
  ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

在现代高性能计算平台（HPC）中，通常采用**HSM + TEE**的组合架构
。HSM作为最高等级的信任锚点，负责存储根密钥（Root
Keys）并处理高敏感度的加密操作；而TEE则利用其高性能处理大吞吐量的加解密请求（如全盘加密）和复杂的安全业务逻辑，TEE通过安全通道请求HSM进行密钥派生或签名操作，从而实现性能与安全性的平衡
。

## **3. 车载安全启动（Secure Boot）深度剖析**

安全启动不仅仅是一个功能，而是一个贯穿系统上电初始化的完整流程。其核心逻辑是"先验证，后执行"（Verify
then Execute）。

### **3.1 启动流程架构（以Qualcomm Snapdragon为例）**

高通平台的启动流程是车载座舱（Cockpit）和智驾（ADAS）域控制器的典型代表。其启动链涉及多个阶段的加载器，每一级都必须验证下一级的签名
。

1.  **PBL (Primary Boot Loader):** 运行在Mask ROM中。

    -   **动作：**
        > 初始化PBL堆栈，加载PBL配置。从引导介质（如UFS）读取XBL（eXtensible
        > Boot Loader）的头部。

    -   **验证：**
        > 计算XBL头部的哈希，验证签名。如果成功，加载XBL到内部SRAM或DDR（如果已初始化）。PBL通常运行在ARM的EL3（Secure
        > Monitor）层级。

2.  **XBL (Secondary Boot Loader):** 相当于U-Boot的SPL阶段。

    -   **动作：**
        > 初始化DDR、时钟、PMIC等核心硬件。加载并验证**QTEE**（Qualcomm
        > Trusted Execution
        > Environment）镜像。这是一个关键步骤，意味着TEE环境在富操作系统（Rich
        > OS）启动之前就已经建立 。

    -   **验证：**
        > XBL验证QTEE镜像的签名，并跳转执行QTEE初始化。随后加载UEFI或ABL（Android
        > Bootloader）。

3.  **UEFI / ABL:** 标准的统一可扩展固件接口或Android专用加载器。

    -   **动作：**
        > 提供标准化的启动环境，支持Fastboot协议。加载Linux内核（boot.img）和Ramdisk。

    -   **验证：** 这一阶段通常开始涉及\*\*Android Verified Boot
        > (AVB)\*\*的逻辑。ABL读取vbmeta分区，验证内核和设备树（Device
        > Tree）的完整性 。

4.  **Hypervisor (管理程序):**
    > 在虚拟化环境中，XBL或UEFI会加载Hypervisor（如QNX
    > Hypervisor或Gunyah）。

    -   **验证：**
        > Hypervisor被验证后启动，随后Hypervisor负责加载和验证各个虚拟机（Guest
        > VMs）的内核镜像 。

### **3.2 证书链与密钥管理**

在这一流程中，密钥管理至关重要。通常存在两条平行的证书链：

-   **SoC厂商链：**
    > 用于签名PBL、XBL等底层固件。根密钥由SoC厂商（如Qualcomm,
    > NXP）持有，OEM在产线将公钥哈希烧录到eFuse。

-   **OEM链：**
    > 用于签名UEFI/ABL、Kernel、System分区。OEM生成自己的密钥对，并将OEM根公钥的哈希包含在SoC厂商签名的证书中（Key
    > Provisioning），或者通过特定的密钥注入流程写入安全存储 。

### **3.3 故障处理与A/B分区机制**

车载系统对可用性有极高要求。如果安全启动验证失败，车辆不能简单地"变砖"。

-   **A/B分区（Slot A/B）：**
    > 现代车载OS普遍采用A/B分区策略。系统包含两套完全独立的启动分区（boot_a,
    > boot_b, system_a, system_b等）。

-   **回退机制：**
    > 引导加载程序维护一些元数据（Metadata），记录当前活跃的插槽（Slot）。如果Slot
    > A验证失败或启动后看门狗（Watchdog）复位次数过多，引导加载程序会自动标记Slot
    > A为"不可启动"（Unbootable），并尝试切换到Slot B进行启动 。

-   **AVB集成：**
    > AVB的元数据（VBMeta）也区分A/B槽位，确保验证数据与当前尝试启动的固件版本匹配。

## **4. Android Verified Boot (AVB) 2.0 深度研究**

AVB 2.0是Google推出的参考验证启动实现，不仅用于Android手机，也是Android
Automotive OS (AAOS)
的核心安全机制。它将信任链从Bootloader延伸到了用户空间的所有分区 。

### **4.1 VBMeta 结构与描述符**

AVB的核心是**VBMeta结构体**（VBMeta
Struct）。这个数据结构既可以独立存在于vbmeta分区，也可以作为页脚（Footer）附加在其他镜像末尾。VBMeta结构体经过密码学签名，是引导加载程序验证后续所有组件的信任源
。

VBMeta包含多种类型的**描述符（Descriptors）**：

1.  **哈希描述符（Hash Descriptor）：**

    -   **适用对象：** boot (Kernel + Ramdisk), dtbo, recovery
        > 等会被一次性加载到内存的小分区。

    -   **机制：**
        > 描述符中包含了整个分区镜像的SHA-256或SHA-512哈希值（以及盐值Salt）。Bootloader在加载这些分区时，实时计算哈希并与描述符中的值比对
        > 。

2.  **哈希树描述符（Hashtree Descriptor）：**

    -   **适用对象：** system, vendor, product 等巨大的文件系统分区。

    -   **挑战：**
        > 无法在启动时读取整个2GB+的分区进行哈希计算，否则启动时间将无法接受。

    -   **机制（dm-verity）：** 采用Merkle
        > Tree结构。文件系统被划分为4KB的块（Block）。底层块的哈希向上层层合并，最终生成一个**根哈希（Root
        > Hash）**。VBMeta中仅存储这个根哈希、盐值和树的偏移量。

    -   **运行时验证：**
        > 内核的dm-verity驱动在读取每个数据块时，实时计算其哈希并验证至根哈希。如果数据块被篡改，哈希验证失败，内核将向应用层返回I/O错误，从而阻止恶意代码的加载或执行
        > 。

3.  **链式分区描述符（Chain Partition Descriptor）：**

    -   **目的：**
        > 实现权限委派（Delegation）。例如，system分区由Google/OEM签名，而vendor分区由Tier-1或SoC厂商签名。

    -   **机制：**
        > 主VBMeta不直接包含vendor分区的哈希，而是包含指向vbmeta_vendor的公钥和签名验证信息。Bootloader在验证主VBMeta后，加载并验证vbmeta_vendor，再由后者验证vendor分区
        > 。

### **4.2 防回滚保护（Rollback Protection）与RPMB**

AVB通过\*\*回滚索引（Rollback
Index）\*\*防止设备降级到旧的、包含已知漏洞的固件版本。

-   **流程：**

    -   每个VBMeta镜像头包含一个整数形式的Rollback Index。

    -   设备在安全存储中维护一个允许的最小Rollback Index值。

    -   Bootloader在验证签名成功后，会检查镜像中的Index是否大于或等于存储中的值。如果小于，则拒绝启动
        > 。

    -   当系统成功更新并启动新版本后，OS会调用TEE服务更新安全存储中的Index值。

-   **RPMB (Replay Protected Memory Block) 的角色：**

    -   在eMMC/UFS存储设备中，RPMB是一个特殊的硬件分区。

    -   **鉴权机制：**
        > 对RPMB的读写操作必须经过HMAC-SHA256签名认证。签名密钥（RPMB
        > Key）是在生产阶段由TEE派生并写入存储控制器的，外部OS无法获知。

    -   **抗重放：** 写入命令包含写入计数器（Write
        > Counter），存储控制器会验证计数器的单调性，防止攻击者通过重放旧的写入命令来篡改回滚索引
        > 。这是TEE与存储设备之间的硬件级安全协议。

### **4.3 libavb 与 Bootloader 集成**

libavb 是Google提供的C语言库，旨在方便SoC厂商将其集成到U-Boot或UEFI中。

-   **平台抽象层：** 集成者需要实现 AvbOps 结构体中的函数，如
    > read_from_partition(), read_rollback_index(),
    > write_rollback_index(), get_unique_guid_for_partition() 等 。

-   **内核命令行：** 验证通过后，libavb
    > 会协助构建传递给Linux内核的命令行参数（cmdline），其中包括
    > dm-verity 的配置参数（如 dm=\"1 vroot none\...
    > root_hash=\...\"）。这确保了内核挂载根文件系统时使用的是经过Bootloader验证的参数
    > 。

## **5. 可信执行环境（TEE）架构与实现**

TEE为车载系统提供了一个与富操作系统（REE）隔离的安全区域，用于处理高敏感数据（密钥、生物特征、支付信息）。

### **5.1 TrustZone 硬件隔离机制**

基于ARM架构的TEE（如OP-TEE, QTEE）依赖于TrustZone扩展。

-   **NS位（Non-Secure Bit）：** 系统总线上的每个读写事务都携带NS位。

-   **TZASC (TrustZone Address Space Controller)：**
    > 内存控制器根据NS位对DRAM进行分区。被标记为安全的内存区域，如果收到NS=1（来自REE）的访问请求，硬件将直接拒绝并可能触发异常
    > 。

-   **异常级别：** TEE OS通常运行在S-EL1（Secure Exception Level
    > 1），受信任应用（TA）运行在S-EL0。而Android/Linux内核运行在NS-EL1。EL3（Secure
    > Monitor）负责在两个世界之间进行上下文切换（World Switch）。

### **5.2 TEE 软件栈：OP-TEE 与 QTEE**

-   **QTEE (Qualcomm TEE):**
    > 高通的专有TEE解决方案。它深度集成在高通的启动链中，在XBL阶段被加载。QTEE提供了广泛的专有API，用于支持Widevine
    > DRM、Gatekeeper、Keymaster等，并能控制SoC内部的加密引擎（Crypto
    > Engine）和熔丝块（QFPROM）。

-   **OP-TEE (Open Portable TEE):** NXP i.MX系列、TI
    > Jacinto等平台常用的开源TEE。

    -   **架构：** 包含运行在安全世界的OP-TEE OS
        > core，以及运行在Linux用户空间的 tee-supplicant（守护进程）和
        > libteec（客户端库）。

    -   **通信：** REE侧应用调用 libteec -\> 内核驱动 optee.ko -\> 执行
        > SMC 指令 -\> CPU陷入EL3 -\> 切换到OP-TEE OS处理请求 -\>
        > 返回结果 。

### **5.3 Android KeyMint (原Keymaster) HAL**

在Android系统中，密钥的生成、存储和使用是通过KeyMint
HAL抽象的，而其真实实现位于TEE中。

-   **密钥Blob：**
    > Android框架层并不直接持有私钥。KeyMint即使生成了密钥，也会将其加密后返回给Android（称为Key
    > Blob）。加密密钥（Key Encryption
    > Key）仅存在于TEE内部，从未暴露给Android。

-   **硬件强制的密钥使用控制：**
    > TEE在解密并使用私钥前，会验证请求是否符合密钥的使用策略（如：是否仅限指纹认证后使用？是否仅限签名操作？）。这意味着即使Android
    > OS被Root，攻击者也无法滥用密钥，因为TEE会拒绝不符合策略的操作请求
    > 。

-   **认证（Attestation）：**
    > KeyMint支持密钥认证功能。TEE可以使用预置的设备私钥（Attestation
    > Key）对存储在TEE中的密钥属性进行签名。远程服务器验证该签名链，从而确信该密钥确实受硬件TEE保护，而非软件模拟
    > 。

## **6. 虚拟化环境下的安全启动与vTEE**

随着座舱域控制器采用虚拟化技术在一颗SoC上同时运行仪表（QNX/Linux）和IVI（Android），安全架构变得更加复杂。Hypervisor（管理程序）成为新的信任锚点。

### **6.1 Hypervisor 的安全启动**

Hypervisor本身必须作为安全启动链的一环被验证。例如，在Qualcomm
SA8155/8295平台上，XBL负责加载并验证Gunyah Hypervisor或QNX
Hypervisor的镜像
。如果Hypervisor被篡改，它不仅能控制所有虚拟机，还能拦截所有硬件访问。

### **6.2 Guest OS（虚拟机）的验证机制**

Hypervisor启动后，负责启动Guest
VMs。此时，Hypervisor扮演了Bootloader的角色，必须验证Guest OS的完整性。

-   **vUEFI (Virtual UEFI):** QNX Hypervisor等提供虚拟的UEFI环境。Guest
    > OS（如Android）看到的就像是在物理机上启动一样，其ABL由vUEFI加载。vUEFI可以配置Secure
    > Boot策略，验证Guest Kernel的签名 。

-   **直接加载验证 (Direct Load):** 一些Hypervisor（如OpenSynergy
    > COQOS）支持直接加载Guest内核镜像。在这种模式下，Hypervisor直接解析vbmeta，计算Guest镜像哈希并与VBMeta比对，实现了虚拟化的AVB流程
    > 。

-   **AVF (Android Virtualization Framework) 与 pKVM:**
    > Google引入了pKVM（Protected KVM），旨在将Guest
    > VM（pVM）的内存与Host OS（Android）隔离。即使Host
    > OS内核被攻破，也无法读取pVM的内存。pKVM利用Stage-2页表映射机制，并在pVM启动时由pvmfw（Protected
    > VM Firmware）验证Payload的签名 。

### **6.3 虚拟化TEE (vTEE)**

多个虚拟机可能都需要访问TEE服务（例如仪表需要解密地图数据，IVI需要DRM）。但物理TEE只有一个。

-   **TrustZone Mediator:**
    > Hypervisor中包含一个中介组件（Mediator），负责拦截Guest
    > VM发出的SMC指令。

-   **上下文切换：**
    > Mediator将虚拟机的ID（VMID）附加到请求中，转发给物理TEE。物理TEE（如支持虚拟化的OP-TEE）为每个VMID维护独立的会话上下文和安全存储区域，确保VM
    > A无法访问VM B的密钥 。

## **7. 远程签名（Remote Signing）与UDS认证**

为了满足供应链安全和合规性（UN
R155/R156），所有的车载软件必须在发布前进行签名。由于开发人员众多且分散，私钥不能存储在本地开发机上，必须采用远程签名架构。

### **7.1 CI/CD 集成的远程签名架构**

现代汽车软件开发采用持续集成/持续部署（CI/CD）流程。

1.  **构建（Build）：** Jenkins/GitLab CI服务器编译生成固件镜像。

2.  **哈希计算（Hashing）：**
    > 构建服务器计算镜像的摘要（Hash），而非传输整个镜像。

3.  **签名请求：**
    > 构建服务器通过mTLS（双向认证TLS）向远程签名服务器发送签名请求，附带哈希值和元数据（项目ID、版本号）。

4.  **HSM操作：**
    > 签名服务器验证请求权限（ACL），将哈希发送给后端连接的HSM（如AWS
    > CloudHSM, Azure Dedicated HSM）。

5.  **私钥操作：** HSM在安全边界内使用存储的私钥（如RSA-4096或ECC
    > P-256）对哈希进行签名。私钥从未离开HSM 。

6.  **封装：**
    > 签名结果返回给构建服务器，构建工具（如avbtool）将签名块注入到镜像的Footer或VBMeta中。

### **7.2 云端HSM (Cloud HSM) 的应用**

相比传统的本地HSM设备，云端HSM提供了更高的弹性和易管理性。

-   **AWS CloudHSM / Azure Dedicated HSM:** 提供FIPS 140-2 Level
    > 3认证的单租户硬件实例。OEM拥有完全的密钥控制权（SoC类），云服务商无法访问密钥。

-   **BYOK (Bring Your Own Key):**
    > OEM可以在离线的安全室中生成根密钥，然后安全地导入到云端HSM中，确保密钥生成的熵源可控且未被云端备份
    > 。

### **7.3 UDS服务0x29 (Authentication)**

传统的UDS安全访问（Service 0x27
SecurityAccess）依赖于静态的Seed/Key算法，容易被逆向工程破解 。ISO
14229-1 (2020) 引入了服务0x29（Authentication）以支持基于PKI的双向认证
。

**0x29 认证流程：**

1.  **证书交换：** 诊断仪（Tester）向ECU发送由OEM CA签名的X.509证书。

2.  **验证：**
    > ECU利用内部存储的OEM根公钥验证诊断仪证书的有效性、有效期及权限扩展字段（Role
    > Extension）。

3.  **挑战-响应：** ECU生成一个随机Nonce发送给诊断仪。

4.  **签名：**
    > 诊断仪使用其私钥（存放在诊断仪的HSM或加密狗中）对Nonce进行签名并发回ECU。

5.  **授权：**
    > ECU验证签名。如果成功，根据证书中的角色（如"刷新权限"、"调试权限"）解锁相应的诊断服务。
    > 这种机制彻底解决了密钥泄露和权限细分的问题 。

## **8. 合规性映射：ISO/SAE 21434 与 UN R155**

上述技术是满足汽车网络安全法规的必要手段。

  ------------------------------------------------------------------------------------------------------------------------------
  **法规/标准条款**      **对应技术实现**      **详细说明**
  ---------------------- --------------------- ---------------------------------------------------------------------------------
  **UN R155 Annex 5      **Secure Boot, AVB**  法规要求防止"通过未经授权的软件更新损害车辆"。安全启动是直接的技术控制措施 。
  (Mitigation to cyber                         
  threats)**                                   

  **ISO/SAE 21434        **Remote Signing**    验证软件的真实性和完整性。远程签名确保了发布软件的可追溯性和防篡改。
  (Verification)**                             

  **ISO/SAE 21434        **TEE / HSM**         需求如"保护加密密钥的机密性"直接映射到使用TEE或HSM进行密钥存储。
  (Security Concept)**                         

  **UN R156 (SUMS)**     **Rollback Protection 软件更新管理系统要求防止回滚到不安全版本。RPMB和eFuse提供了硬件强制的防回滚能力
                         (RPMB)**              。

  **TARA (Threat         **CAL (Cybersecurity  TARA分析若判定某ECU面临高风险（如CAL
  Analysis and Risk      Assurance Level)**    4），则必须实施基于硬件信任根的强安全启动，而非纯软件校验 。
  Assessment)**                                
  ------------------------------------------------------------------------------------------------------------------------------

## **9. 未来趋势：后量子密码学 (Post-Quantum Cryptography)**

随着量子计算的发展，现有的RSA和ECC算法面临被Shor算法破解的风险。由于车辆生命周期长达10-15年，现在的设计必须考虑未来的量子威胁（Store
Now, Decrypt Later）。

### **9.1 LMS 与 XMSS 签名方案**

NIST推荐在固件签名场景中使用**基于哈希的签名算法**（Stateful Hash-Based
Signatures），如**LMS (Leighton-Micali Signature)** 和 **XMSS (Extended
Merkle Signature Scheme)** 。

-   **优势：**
    > 基于SHA-256等哈希函数构建，抗量子计算攻击。验证速度快，代码量小，非常适合Mask
    > ROM和Bootloader环境。

-   **挑战：**
    > 它们是"有状态"的（Stateful）。私钥只能使用有限次，且必须严格管理状态（计数器），防止一次性密钥对被重复使用（会导致私钥泄露）。这要求远程签名基础设施具备极其严格的状态同步机制。

-   **迁移：** 芯片厂商（如ST, Infineon,
    > NXP）已开始在下一代MCU的硬件加速器中集成LMS/XMSS验证逻辑，以实现量子安全的启动链
    > 。

## **结论**

车载系统的安全性构建在从硅片到云端的精密架构之上。安全启动与AVB通过层层递进的签名验证，确保了代码的静态完整性；TEE与HSM通过物理或逻辑的隔离，构筑了运行时密钥与敏感数据的避风港；Hypervisor与vTEE技术则解决了域控架构下多系统共存的安全隔离难题。而远程签名基础设施与UDS
0x29认证服务，则为软件的整个生命周期管理（开发、更新、诊断）提供了可信的控制平面的保障。面对合规性压力与量子计算的远期威胁，这一架构仍在持续演进，向着更敏捷、更具韧性的方向发展。

## **10. 参考文献 (引用列表)**

-   : Embitel. \"TEE based ECU Security for Secure Automotive Systems.\"

-   : Android Open Source Project. \"Android Verified Boot 2.0.\"

-   : GlobalPlatform. \"Realizing Secure Boot.\"

-   : Renesas. \"Introduction about Secure Boot.\"

-   : Android Open Source Project. \"Boot Flow.\"

-   : Qualcomm. \"Boot Flow and Architecture Overview.\"

-   : U-Boot Doc / AOSP AVB README. \"AVB 2.0 details.\"

-   : Arxiv/Research. \"RPMB Implementation.\"

-   : AOSP/Kynetics. \"dm-verity internals.\"

-   : Entrust/AWS. \"Remote Signing Architecture.\"

-   : Qualcomm. \"QTEE and TrustZone.\"

-   : QNX. \"Secure Boot in Hypervisor.\"

-   : OpenSynergy. \"COQOS Hypervisor Datasheet.\"

-   : UNECE. \"Regulation No. 155.\"

-   : Research/Lattice. \"Post-Quantum Cryptography in Automotive.\"

-   : Embitel/GlobalPlatform. \"TEE vs HSM.\"

-   : WhyEngineer/AlefBits. \"UDS Service 0x29.\"
