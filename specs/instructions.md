# Instructions

## keystore definition

请详细介绍andorid系统中的keystore，并进行思维发散，做相关概念的介绍,使用 mermaid 绘制架构、设计、组件、流程等图表并详细说明,保存到./specs/keystore.md


## relation

请介绍下keystore、keymint、指纹、生物认证、keymaster、strongbox、gatekeeper、TEE、keystore2、Key Attestation这些概念，梳理它们之间的关系一节实际业务场景，保存到./specs/relation.md



## Android

介绍下Android系统的架构，使用 mermaid 绘制架构、设计、组件、流程等图表并详细说明,保存到./specs/android.md

## 现状

当前车载系统有如下功能，请结合./specs下文档以及上网搜索，分析有哪些方法可以提升车载系统的整体安全性，包括兼容备份、仲裁等功能,保存到./specs/sec-new.md

tls通信：应用层TLS、native层TLS调safekeyservice进行TLS通信;safekeyservice调中间件;中间件调TT密码模块和安全芯片so


TEE密码模块
6、功能包括设备管理、回话管理、授权管理、秘钥管理、密码服务、安全存储、证书管理、日志管理等。
7、日志管理：
○对重要信息生成日志进行存储，并对日志进行管理；
○重要信息包含调用者的数据输入过程、结果输出过程等关键环节的错误信息、调用者权限管理、访问控制管理等敏感内容；
○日志应支持等级控制；
8、证书管理：生成证书请求，格式符合RFC 2986
9、安全存储：
○应对外提供安全文件的创建、读、写、删除等功能；
○在进行数据读取和写入时， 应建立安全通道和数据保护机制，确保数据的机密性和完整性；
○应在读、写、创建、删除前进行鉴别和授权；
○不同TA的加密密钥应不同，防止TA之间的水平越权访问；
○安全存储区应采用防御技术保证其不被非法获取和篡改。

10、密钥服务○RSA2048/ECC/SM2签名验签
○RSA2048/ECC/SM2加密解密
○AES128/AES192/AES256/SM4加密解密，至少支持ECB/CBC模式
○支持MD5/SM3/SHA256
○支持AES128/AES192/AES256/SM4 CMAC算法
○宜支持SHA256HMAC算法
○宜支持KDF算法
○宜具备密钥协商功能

11、密钥管理
○密钥创建：创建对称/非对称密钥，可根据实际情况进行落盘或导出。
○密钥导入：应具备密钥明文和密钥密文的导入功能，如向TEE内导入业务密钥等，并确保密钥导入的安全性
○应具备密钥的访问控制功能，在密钥生成或导入时，密钥使用方应为密钥指定其被授权使用的方式或目的，包括但不限于加密、解密、签名、身份验证等。在密钥使用时，由TEE执行授权校验，避免密钥被未经授权的方式使用。
○应支持非对称密码算法的公钥及证书从TEE中导出的功能，确保其他类密钥的明文不出TEE，不应支持私钥的导出功能。
○密钥获取：获取已落盘的密钥的信息，其中，对称密钥和非对称密钥的私钥无法导出。

12、授权管理
○初始化：提供对设备进行初始化功能，初始化之前密钥管理、密钥服务、文件管理、证书管理等功能均不可用。此功能只应出现在产测版本，不应在SOP版本提供此功能。
○具备身份信息写入与身份信息存储功能；
○用户身份信息应被安全的存储在RMPB或安全存储区中，并提供身份信息防泄露机制；
○身份信息的注册与校验过程应在TEE内部进行，确保注册与校验过程的安全性，并提供防暴力破解机制，包括但不限于设置错误验证次数阈值、基于时间惩罚的验证锁定等；
○宜提供达到错误验证次数阈值后的身份信息删除功能，以防止恶意攻击。


13、会话管理
○打开、关闭会话，可多次打开关闭，支持多进程多线程

14、设备管理
○打开、关闭设备，可多次打开关闭，支持多进程多线程
○获取设备SN等硬件信息，设备SN每个设备唯一，由4位型号标识符和自定义12位标识符组成。
○获取密码模块软件版本号，每次升级变更
○版本升级，宜支持AB升级，应具备防止版本回退功能


中间件
1、支持主流的HSM、TEE、软件密码模块通用接口的中间层，方便上层应用统一调用，并对密码设备进行调度管理，提高可靠性。
2、同时支持android和linux系统；在Linux平台，向下兼容安全芯片接口，同时兼容标准GP接口，及PKCS11接口，向上提供基于GMT 0018修改的SDF接口，同时支持openssl 的engine库，供应用方进行双向认证的能力；在Android平台，向下兼容安全芯片接口，兼容标准GP接口，及PKCS11接口。向上提供基于GMT 0018修改的SDF接口，应用方会通过指定套件的方式调用SDF接口进行双向认证。
3、外部接口，对于密码管理接口，参考GMT 0018进行标准化接口设计；对于linux下双向https接口，参考openssl engine的方式进行设计；
4、信息安全中间件为证书管理、数据解密、数据存储、密钥运算等提供密钥的集中管理，支持RSA、AES等密钥的安全保护，证书的存取，自定义数据存取等。
5、可靠存储
	应尽可能提供安全可信环境进行存储，安全可信环境包括但不限于HSM/TEE/SE等
6、数据加密
若设备具备安全可信环境，则应提供多层密钥结构进行数据的安全存储，使用安全可信环境的密钥加密密钥对数据进行加密存储。
7、密码运算
a)支持RSA2048\ECC\AES128\AES256\SHA256\SHA256-HMAC等国际通用算法；
b)AES算法模式至少支持ECB/CBC/GCM
c)国内车型扩展支持国密SM2/SM4/SM3
d)SM4算法模式至少支持ECB/CBC
8、隔离运算
长期密钥在生命周期内不可出安全存储区，以防泄露。
9、标准接口
a)提供SDF接口，在linux侧接入OpenSSL，在Android侧接入BroingSSL；
b)提供密钥管理、安全存储类接口；
c)提供生成P10证书的接口；
d)具体可参考DX8155博世TEE服务接口；

Safekey
Safekeyservice封装了中间件的调用，对外支持密码学算法；产线秘钥、证书的灌装；运行在native层的服务safekeyservice

framework层封装SafeKeyManager通过Binder调native层safekeyservice
 IBinder b = ServiceManager.getService("safekeyservice");
        mService = ISafeKeyManagerService.Stub.asInterface(b);

应用层使用framework层封装的SafeKeyManager
mSafeKeyManager = new SafeKeyManager(context);



        OkHttpClient client = new OkHttpClient.Builder()
                .connectTimeout(30, java.util.concurrent.TimeUnit.SECONDS)  // 连接超时时间
                .readTimeout(30, java.util.concurrent.TimeUnit.SECONDS)     // 读取超时时间
                .writeTimeout(30, java.util.concurrent.TimeUnit.SECOND

内部通过SafeKeyManager获取证书，签名验签
