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
* 使用用户的默认配置创建一个签名密钥。这里我们使用P_RSA2048SHA1这个配置而不是默认的。参数UNK说明这个密钥是不可复制的。名称是mySigningKEy。ASYM_RESTRICTED_SIGNING_KEY这个参数表明这个密钥是一个签名密钥。同时我们还使用一个非常容易验证通过的Policy和一个空的口令。
```
Tss2_Key_Create(context, // pass in the context I just created
"P_RSA2048SHA1/UNK/mySigningKey", // non-duplicableRSA2048
ASYM_RESTRICTED_SIGNING_KEY, // signing key
TSS2_POLICY_TRIVIAL, // trivially policy
TSS2_AUTH_NULL); // the password is NULL
```
* 使用密钥对“Hello World”签名时，首先要使用OpenSSL软件库来对它做哈希。
```
TSS2_SIZED_BUFFER myHash;
myHash.size=20
myHash.buffer=calloc(20,1);
SHA1("Hello World",sizeof("Hello World"),myHash.buffer);
```
* 签名命令将会返回所有用于验签的信息。因为密钥是刚刚创建的，所以密钥的证书是空的。
```
TSS2_SIZED_BUFFER signature, publicKey,certificate;
Tss2_Key_Sign(context, // pass in the context
"P_RSA2048SHA1/UNK/mySigningKey", // the signing key
&myHash,
&signature,
&publicKey,
&certificate);
```
* 通常情况下我们应该保存签名的结果，但是我们在这个示例中直接做验证签名的操作。
```
if (TSS_SUCCESS!=Tss2_Key_Verify(context ,&signature,
&publicKey,&myHash) )
{
printf("The command failed signature verification\n");
}
else printf("The command succeeded\n");
```
* 销毁之前申请的缓冲区，示例程序就算完成了。
```
free(myHash.buffer);
free(signature.buffer);
free(publicKey.buffer);
/* I don’t have to free the certificate buffer, because
it was empty */
Tss2_Context_Finalize(context);
```

我们很容易看出来这个示例有点太简单了。具体来说就是，密钥在创建时不需要任何认证。接下来我们将介绍如果密钥需要认证时，应该怎么做。

所有的FAPI函数都假设密钥仅仅使用Policy做认证。如果密钥需要口令来认证，那么密钥将被设置一个口令，同时还会使用TPM2_PolicyAuthValue命令创建一个Policy。默认配置中的TSS2_POLICY_AUTHVALUE就是这样做的。但是，如何满足这个Policy是一个大问题。

Policy相关的命令有两类。一类是需要和TPM外部通信的命令：
* PolicyPassword：要求输入口令。
* PolicyAuthValue：要求输入口令。
* PolicySecret：需要输入口令。
* PolicyNV：需要输入口令。
* PolicyOR：需要在可选列表中做一个选择。
* PolicyAuthorize：需要选择一个授权过的选项。
* PolicySigned：需要一个来自某个设备的签名。

另外一种是需要和TPM外部打交道的命令：
* PolicyPCR：命令会检查TPM内部的PCR值。
* PolicyLocality：检查命令对应的Locality。
* PolicyCounterTimer：检查TPM内部的计数器。
* PolicyCommandCode：检查是什么命令。
* PolicyCpHash：检查命令和命令相关的参数。
* PolicyNameHash：检查送往TPM内部的对象的名称。
* PolicyDuplicationSelect：检查密钥被复制的目的位置。
* PolicyNVWritten：检查一个NV索引有没有被写过。

许多Policy需要以上两类命令配合使用。如果一个Policy需要上述第二种授权方式，FAPI会负责处理。如果是第一种授权，那用户就需要向FAPI提供相应的参数。

用户向FAPI提供参数是通过回调机制来实现的。你必须首先在程序中注册这些回调函数，这样以来FAPI就是知道要求口令，选择，或者签名时应该调用什么样的函数来和用户交互。一种有三种类型的回调函数，定义如下：
* TSS2_PolicyAuthCallback：用于需要输入口令的情况。
* TSS2_PolicyBranchSelectionCallback：用于用户在执行PolicyOR或者TPM_PolicyAuthorize命令时选择其中一个Policy。
* TSS2_PolicySignatureCallback：用于当一个Policy需要用户输入一个签名的情况。

上述的第一个回调是最简单的。当注册Context之后，就需要创建一个回调函数，当FAPI需要要求用户输入密码时就会调用该函数。FAPI会通过回调函数向程序提供需要授权的对象的描述，然后申请输入授权数据。FAPI还负责对授权数据做加盐和HMAC操作。用户需要做两件事情：一是创建一个要求用户输入口令得函数，然后注册它，这样FAPI就可以调用了。

如下是一个简单的口令处理函数：
```
myPasswordHandler (TSS2_CONTEXT context,
void *userData,
char const *description,
TSS2_SIZED_BUFFER *auth)
{
/* Here the program asks for the password in some application specific
way. It then puts the result into the auth variable. */
return;
}
```

下面这个函数用于向FAPI注册你的回调函数：
```
Tss2_SetPolicyAuthCallback(context, TSS2_PolicyAuthCallback, NULL);
```

对于其他类型的回调函数来说，其创建和注册过程是类似的。

在写这本书的时候， 使用XML来为命令创建Policy的规范还没有开始写，不过很可有可能在2014年发布这样的规范。有一件事情可以确认：硬件的OEM厂商可以提供一个包含有自己回调函数的软件库。这时候，回调函数会在Policy中注册而不是在程序中，因此开发者就不用重新再写了。同样的，Policy允许使用这样的软件库来注册这些回调函数。这种情况下就不需要再注册任何回调函数了。

## System API
前面我们已经提到，TPM2.0的SAPI这一层软件使用相当于用C语言编写软件。SAPI实现了TPM2.0所有的功能。当我们在描述底层接口时经常会提起这一点，底层软件接口给软件工程师提供了所有他们把自己吊死所需要的绳索。SAPI就像是C语言一样，它是一个功能强大的工具，但是需要非常专业的只是来用好它。

SAPI规范可以在以下网址找到：www.trustedcomputinggroup.org/developers/software_stack。 SAPI的主要设计目标如下：
* 提供所有TPM功能的访问接口。
* 可以在尽可能多的平台上使用，从高度嵌入式，内存受限的环境到多核的服务器上都可以使用。为了支持较小的应用，SAPI的代码需要考虑很多，从而可以让内存使用最小化或者提供最小化的选项。
* 在提供所有功能的前提下，尽可能让程序员的工作容易。
* 支持同步和异步调用。
* SAPI的实现本身不需要申请任何内存。同时SAPI的使用者要负责申请所有SAPI使用的内存。

SAPI提供了四组命令：TPM命令上下文的申请，命令准备，命令执行，命令完成。这一节将描述这几组命令。在命令准备，执行和完成这几组命令中会有一些辅助函数被调用，这些函数应用于所有TPM规范第三部分定义的所有TPM命令，其他的函数则是因命令不同而不同。

首先我们会概括性地介绍每一组命令。介绍这些命令时，我们会列出TPM2_GetTestResult示例的代码片段。最后我们将这些代码片段组合起来，并使用三种方法来执行TPM2_GetTestResult这个命令：单独的一次调用，异步调用，同步的多次调用。SAPI函数的代码示例要求了解会话，授权，加解密等概念，但是这些概念都推后到第13和17章中介绍。只有当你真正理解了这些TPM功能之后，SAPI的这些函数才有意义。这一章的最后会简单介绍一下SAPI代码中附带的测试程序。

### 命令上下文申请函数
下面要介绍的函数用于申请SAPI命令上下文数据结构的内存空间。这些难懂的数据结构用于维护TPM2.0命令执行时的状态数据。

Tss2_Sys_GetContextSize这个函数用于决定SAPI上下文数据结构需要多少内存空间。这个命令会返回内存空间的大小能够满足执行任何TPM2.0规范第三部分中描述的命令。或者调用者也可以提供最大的命令和响应大小，这个函数会计算所需的上下文大小。

Tss2_Sys_Initialize用于初始化SAPI的上下文。它需要如下四个参数：一个指向足够用于上下文的内存区域的指针；Tss2_Sys_GetContextSize返回的上下文大小；指向TCTI上下文的指针，这个上下文定义了发送命令和接收命令响应的方法；最后还有SAPI版本信息。
```
注：以下代码中的一个提示：rval是 return value的缩写，它是一个无符号的32位整数。后续的代码示例将重复使用rval。
```
如下是一个创建和初始化系统上下文结构的代码示例。
```
函数要求返回一个TSS2_SYS_CONTEXT的指针。这个结构体的定义如下：
typedef struct _TSS2_SYS_OPAQUE_CONTEXT_BLOB TSS2_SYS_CONTEXT;

```

```
//
// Allocates space for and initializes system
// context structure.
//
// Returns:
// ptr to system context, if successful
// NULL pointer, if not successful.
//
TSS2_SYS_CONTEXT *InitSysContext(
UINT16 maxCommandSize,
TSS2_TCTI_CONTEXT *tctiContext,
TSS2_ABI_VERSION *abiVersion
)
UINT32 contextSize;
TSS2_RC rval;
TSS2_SYS_CONTEXT *sysContext;
// Get the size needed for system context structure.
contextSize = Tss2_Sys_GetContextSize( maxCommandSize );
// Allocate the space for the system context structure.
sysContext = malloc( contextSize );
if( sysContext != 0 )
{
// Initialize the system context structure.
rval = Tss2_Sys_Initialize( sysContext,
contextSize, tctiContext, abiVersion );
if( rval == TSS2_RC_SUCCESS )
return sysContext;
else
return 0;
}
else
{
return 0;
}
}
```


### 命令准备函数
### 命令执行函数
### 命令完成函数
### 简单的代码示例
### SAPI测试代码
前面出现的GetTestResult作为一个测试被包含在SAPI测试代码中。

## TCTI
## TPM访问代理（TPM Access Broker）
## 资源管理器
## 设备驱动
## 总结
