# TPM软件栈
这本书的初衷是介绍TPM2.0设备。但是若果没有配套的软件，TPM就是一个满载燃料却没有司机的汽车，它有巨大的潜力，可是哪里都去不了。同样也是为了本书剩余部分做铺垫，这一章给大家介绍一下TPM的“司机”：TSS。理解这一章的内容将有助于你学习本书后续的示例代码。

TSS是TCG的软件标准，TSS包含几个可以移植的软件层，这样就允许应用软件使用其中间层。基于TSS编写的软件应该可以运行在任何实现TSS标准的系统上。这一章介绍了TSS的软件分层，其中重点描述SAPI和FAPI，其他层则在概述中描述。

## 软件栈概览
TSS包含以下由高到低的几层软件：FAPI，ESAPI，SAPI，TCTI（TPM Command Transmission Interface），TAB（TPM Access Broker），RM（Resource Manager），和设备驱动。

大多数的用户层引用程序基于FAPI开发就可以了，因为FAPI实现了TPM百分之八十的常用应用场景。使用这一层开发应用就像是使用JAVA，C#等高级语言开发应用一样方便。

往下一层是ESAPI，它需要你对TPM了解很深，但是同时提供了会话管理以及加解密的辅助功能。这有点像使用C++开发应用程序。写这本书是ESAPI的规范还在制定中，这一章不做介绍。

应用程序也可以直接基于SAPI这一层，但这需要你对TPM了如指掌。这就像是使用C语言编写应用程序，而不是用高级语言。它提供了TPM的所有功能，但是要想用好它你必须对TPM有很深的理解。

TCTI层用于向TPM发送命令并接收TPM对命令的响应。应用可以直接通过TCTI发送命令的数据流并解析接收到的响应数据流。这就像是使用汇编语言来编写应用程序。

TAB这一层主要负责多线程环境下TPM资源的同步。也就是说它允许多个线程同时访问TPM而不发生冲突。

因为TPM内部的存储资源非常有限，所以需要一个资源管理器RM，它的原理于虚拟内存管理类似，它可以将TPM对象和会话换进换出TPM。

最后一层就是设备驱动，它主要是控制通信外设与TPM互相传输数据。如果你愿意的话，直接调用设备驱动接口来编写应用程序也是可以的，当然这就像是你用二进制数据编写程序一样。

图7-1描述了TSS的组件构成。主要注意以下几点：
* 尽管通常情况下，对于应用来说只有一个可用的TPM设备，但是有多个TPM设备也是可以的。其他的TPM设备可以是软件实现的TPM，比如微软的TPM模拟器；也可以是通过网络来远程访问的TPM，TPM设备有这样的远程管理功能。
* 通常来说，SAPI以上的软件是线程相关的。
* SAPI以下的组件是与TPM硬件相关的。
* 尽管图7-1没有体现这一点，实际上TCTI可以作为RM和设备驱动的接口，这时候TCTI在软件栈中有多个层次的角色。
* 到现在位置，TAB和RM最常见的实现方式是为一个模块实现。

图7-1

接下来分别介绍TSS的每一层内容。

## Feature API
TSS的FAPI目标是让用户更容易地使用TPM2.0最常用的功能。因此，FAPI不能使用使用TPM的一些特殊功能。

在设计FAPI时，设计者希望80%的应用程序仅仅使用FAPI这一层就能满足要求，而不用使用其他的TSS API。同时也尽可能让用户较少地调用API函数，以及定义最少的参数。

实现上述构想的一种实现方式就是使用一个配置文件，这个配置文件中定义了用户对算法，密钥大小，加密模式，签名模式等的默认配置；用户在创建密钥的时候就可以使用这些默认的配置。它假设用户希望选择一组互相匹配的密码，用户可以设置是否使用这些默认的配置。有时候用户可能希望选择一个配置文件，当然这也是可以的。但是通常情况下用户总是会选择默认的配置。FAPI在发布的时候总是附带一个预先设置好的配置文件，配置文件总是包含最常用的配置。举例如下：
* P_RSA2048SHA1这个配置使用RSA2048位的非对称密钥来签名，签名过程遵循PKCS1v1.5规范。哈希算法使用SHA1，对称加密使用AES128的CFB模式。
* P_RSA2048SHA256这个配置同样使用RSA2048位非对称密钥类签名，签名过程遵循PKCSv1.5规范。哈希算法使用SHA256，对称加密使用AES128的CFB模式
* P_ECCP256这个配置的签名机制时ECDSA，密钥使用NIST ECC素数域256比特非对称密钥。

配置文件中的路径描述用于FAPI查找，密钥，Policies，NV，和其他的TPM对象和资源实体。路径的基本结构如下：
<Profile name> / <Hierarchy> / <Object Ancestor> / key tree

如果在使用FAPI时忽略了配置文件名称，那它将使用默认的配置。如果没有设置组织（Hierarchy），那么将使用默认的存储组织架构，存储组织架构叫做H_S，背书组织架构（Endorsement Hierarchy）是H_E，平台组织架构（Platform Hierarchy）是H_P。一个对象的父对象可以是如下的值：
* SNK：不可迁移密钥的系统父对象。
* SDK：可迁移密钥的系统父对象。
* UNK：不可迁移密钥的用户父对象。
* UDK：可迁移密钥的用户父对象。
* NV：用于NV的创建。
* Policy：用于Policy对象。

密钥树就是由一系列用/隔开的父密钥和子密钥组成。注意这个路径是不区分大小写的。

下面我们来看一些示例，假设用户使用P_RSA2048SHA1，所有以下的路径都是相等的：
* P_RSA2048SHA1/H_S/SNK/myVPNkey
* H_S/SNK/myVPNkey
* SNK/myVPNkey
* P_RSA2048SHA1/H_S/SNK/MYVPNKEY
* H_S/SNK/MYVPNKEY
* SNK/MYVPNKEY

一个父对象为用户备份密钥的ECC P-256 NIST 签名密钥可以用如下的路径标识：
* P_ECCP256/UDK/backupStorageKey/mySigningKey

FAPI还包含一些默认资源实体的名称：

密钥：
* ASYM_STORAGE_KEY：一个用于存储密钥或者数据的非对称密钥。
* EK：一个含有证书的背书密钥，它用户证明（或者在一个操作中证明其他的密钥）它属于一个真实的TPM。
* ASYM_RESTRICTED_SIGNING_KEY：一个类似于TPM1.2中的AIK，但是它可以用于对TPM之外的数据签名。
* HMAC_KEY：一个不受限制的对称密钥。它的主要用途就是用于HMAC对非TPM产生的哈希做签名。

NV：
* NV_MEMORY：普通的NV内存。
* NV_BITFIELD：一个64位的位域。
* NV_COUNTER：一个64位的计数器。
* NV_PCR：使用哈希算法模板的NV_PCR。
* NV_TEMP_READ_DISABLE：在一次启动周期内禁止读操作。

标准的Policy和认证：
* TSS2_POLICY_NULL：一个永远都不会通过的NULL Policy（空缓冲区）。
* TSS2_AUTH_NULL：一个空口令。
* TSS2_POLICY_AUTHVALUE：指向一个对象的授权数据。
* TSS2_POLICY_SECRET_EH：指向背书组织架构的授权数据。
* TSS2_POLICY_SECRET_SH：指向存储组织架构的授权数据。
* TSS2_POLICY_SECRET_PH：指向平台组织架构的授权数据。
* TSS2_POLICY_SECRET_DA：指向字典攻击handle的授权数据。
* TSS2_POLICY_SECRET_TRIVIAL：指向一个全0的policy。这个policy很容易通过，因为每一个policy会话都是由全0开始的。它可以用于通过FAPI创建一个很容易满足policy的对象。

所有使用FAPI创建的对象都是用Policy的授权方式。但是这也不意味着不能使用口令授权：例如在TSS2_POLICY_AUTHVALUE时就可以。但是通常情况下是不会使用口令的。如果一定要使用口令，通常是在加盐的HMAC会话中。

TSS2_SIZED_BUFFER是FAPI经常使用的一个数据结构。这个结构包含两个域：一个size和一个指向缓冲区的指针。size代表了缓冲区的大小：
```
typedef struct { size_t size;
uint8_t *buffer;
} TSS2_SIZED_BUFFER;
```

在开始写程序之前你还需要知道一件事：在程序的开始，你必须首先创建一个context，使用完以后要销毁它。

下面让我们使用FAPI来编写一个创建密钥的示例程序。然后用这个密钥来签名“Hello World”，之后还要验证签名。我们安装以下步骤来做：
* 创建一个Context，告诉这个context使用本地的TPM，也就是设置第二个参数为NULL：
```
TSS2_CONTEXT *context;
Tss2_Context_Intialize(&context, NULL);
```
* 使用用户的默认配置创建一个签名密钥。这里我们使用P_RSA2048SHA1这个配置而不是默认的。参数UNK说明这个密钥是不可复制的。名称是mySigningKEy。
## System API
### 命令上下文申请函数
### 命令准备函数
### 命令执行函数
### 命令完成函数
### 简单的代码示例
### SAPI测试代码
## TCTI
## TPM访问中介（TPM Access Broker）
## 资源管理器
## 设备驱动
## 总结
