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
但是这个数据结构并没有并定义。这样仍然可以工作的原因是TSS2_SYS_CONTEXT这个结构总是通过指针来使用。 使用这种结构体指针的方式相对于void指针有一个好处，那就是在编译时做类型检查。
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

这一组中最后一个函数是Tss2_Sys_Finalize，它用于在释放SAPI上下文数据结构之前清理这些数据。如下是一个使用案例：
```
void TeardownSysContext( TSS2_SYS_CONTEXT *sysContext )
{
if( sysContext != 0 )
{
Tss2_Sys_Finalize(sysContext);
free(sysContext);
}
}
```

```
注：在这个示例中，Tss2_Sys_Finalize是一个不做任何操作的函数，因为SAPI代码不需要它做任何事情。系统上下文的内存在Finalize命令执行以后被释放。
```

### 命令准备函数
第13，17章会介绍到，HMAC计算，命令参数加密，命令响应参数解密等操作是经常在命令执行前后需要做的事情。命令准备函数用于命令数据在发送到TPM设备之前需要做的操作。

为了计算命令的HMAC和加密命令执行参数，命令的参数必须首先被标准化。这个操作本来可以用额外的代码来实现，但是既然SAPI中已经包含了这个功能，SPI的设计者就决定把相关的函数提供给用户使用。这也就是Tss2_Sys_XXXX_Prepare这一系列函数的由来。因为规范第三部分中不同命令的参数各不相同，所以每一个命令都有一个类似的函数。“XXXX”就代表命令的名字。比如说，针对命令TPM2_StartAuthSession的Tss2_Sys_XXXX_Prepare函数就是Tss2_Sys_StartAuthSession_Prepare。以下是为TPM2_GetTestResult做准备的函数调用示例：
```
rval = Tss2_Sys_GetTestResult_Prepare( sysContext );
```

```
注：这个示例中唯一需要输入的参数就是系统上下文的指针，因为TPM2_GetTestResult这个命令不需要输入参数。
```

在调用Tss2_Sys_XXXX_Prepare函数之后，命令数据就被标准化了。为了得到标准化的数据流，还需要调用Tss2_Sys_GetCpParam。这个函数会返回cpBuffer的初始位置，标准化的命令参数字节流，以及cpBuffer的长度。具体的使用方式将在第13，17章中介绍。

计算命令的HMAC还需要另外一个函数Tss2_Sys_GetCommandCode。这个函数会以CPU大小端的方式返回命令代码。在命令的后处理时也会用到这个函数。

Tss2_Sys_GetDecryptParam和Tss2_Sys_SetDecryptParam用于解密会话，17章会详细介绍。现在我们需要知道的是，Tss2_Sys_GetDecryptParam会返回一个指向被加密数据其实位置的指针和参数的大小。这两个值将被用于Tss2_Sys_SetDecryptParam函数，这个函数用于向命令数据流中添加加密参数。

Tss2_Sys_SetCmdAuths函数用于设置命令的授权区域（也叫做授权会话），这将在第13章讨论会话和授权时详细介绍。

### 命令执行函数
这一组函数用于向TPM发送命令和从TPM接收命令响应数据。命令可以被以同步或者异步的方式发送。同步发送又分为两种：3-5个函数调用；或者是完成所有事情的一次调用。支持同步，异步和单次，多次调用主要是为了支持尽可能多的应用架构。

Tss2_Sys_ExecuteAsync是发送命令的最基础方法。它使用TCTI发送函数来发送命令，并尽快返回。如下是一个示例：
```
rval = Tss2_Sys_ExecuteAsync( sysContext );
```

Tss2_Sys_ExecuteFinish是与ExecuteAsync配套的函数。它会调用TCTI函数来接收命令响应数据。这个函数有一个timeout参数用于决定要命令响应的超时时间。下面是一个响应超时设置为20ms的例子：
```
rval = Tss2_Sys_ExecuteFinish( sysContext, 20 );
```
Tss2_Sys_Execute是与Tss2_Sys_ExecuteAsync功能相同的异步方法。在这个函数调用之后就是 Tss2_Sys_ExecuteFinish，它同样是同步的方法，所以相当于无限大的timeout。如下是一个调用示例：
```
rval = Tss2_Sys_Execute( sysContext );
```

这一组中的最后一个函数是 Tss2_Sys_XXXX，他就是针对不同命令的一次调用完成所有任务的函数。这个函数假设对应的操作不需要授权，或者仅仅需要一个简单的口令授权，又或者是HMAC和Policy授权已经计算完成。同样地，规范第三部分中的每个命令对应一个函数。比如说，Tpm2_StartAuthSession这个命令对应的一次性调用函数就是Tss2_Sys_StartAuthSession。这个函数和Tss2_Sys_XXXX_Prepare配合使用就可以完成各种类型的授权。但是这样做的一个有意思的副作用就是命令的参数将会被标准化两次：一次在Tss2_Sys_XXXX_Prepare调用，一次是在一次性的执行函数调用中。这种设计上的妥协是因为一次性的函数需要包含Tss2_Sys_XXXX_Prepare 所具备的功能。下面是一个不需要授权的命令的一次性调用执行示例：
```
rval = Tss2_Sys_GetTestResult( sysContext, 0, &outData, &testResult, 0 );
```

```
注： 这个函数需要以下参数：一个系统上下文结构体指针，一个命令授权信息的指针；两个输出参数，outData和testResult；还有一个命令响应授权数据结构的指针。示例中参数为0的就是命令和命令响应授权数据的指针。对于这个简单的示例而言，授权数据是不需要的，所以传递了空指针。第13章还会介绍它的使用。
```

### 命令完成函数
这一组函数用于TPM命令的后处理。这些后处理包括命令响应的HMAC计算，以及在加密会话中命令响应参数的解密。

Tss2_Sys_GetRpBuffer需要一个指向命令响应参数数据的指针，以及参数数据的大小作为参数。知道这两个参数后，用户就可以计算命令响应的HMAC，然后与命令响应授权区域的HMAC比较。

Tss2_Sys_GetRspAuths用于获取命令响应授权区的数据。比如上面刚刚提到的HMAC。

完成响应数据的验证后，如果命令响应是通过加密会话传输的，就可以进一步通过Tss2_Sys_GetEncryptParam和Tss2_Sys_SetEncryptParam解密响应参数，并将它们插入到响应的数据流中，后续的反标准化（unmarshalling）操作会将它们解析成对应的数据结构。在后面的第17章加解密会话中会详细地介绍这两个函数。

响应参数被解密后，数据流就可以被反标准化了。这个操作使用Tss2_Sys_XXXX_Complete。因为不同的命令有不同的响应参数，所有规范第三部分中的每个命令也对应一个函数。示例如下:
```
rval = Tss2_Sys_GetTestResult_Complete( sysContext, &outData, &testResult );
```

现在为止，所有的SAPI函数就介绍完了。总结下来就是，其中有一些是针对不同TPM命令的，有一些则是通用的。

### 简单的代码示例
下面要介绍的是SAPI代码库中的测试代码，它用三种方式完成TPM2_GetTestResult的调用：一次性调用，同步调用，异步调用。

代码的注释很好的说明了三种调用：
```
注： CheckPassed()是一个用于检查输入参数，也就是上一次调用的返回值，否是0的函数。如果返回值不是0，说明有错误，这个函数会打印错误信息，清理程序相关数据并退出。
```

```
void TestGetTestResult()
{
UINT32 rval;
TPM2B_MAX_BUFFER outData;
TPM_RC testResult;
TSS2_SYS_CONTEXT *systemContext;
printf( "\nGET TEST RESULT TESTS:\n" );
// Initialize the system context structure.
systemContext = InitSysContext( 2000, resMgrTctiContext, &abiVersion );
if( systemContext == 0 )
{
Handle failure, cleanup, and exit.
InitSysContextFailure();
}
test the one-call apI.
//
// First test the one-call interface.
//
rval = Tss2_Sys_GetTestResult( systemContext, 0, &outData, &testResult,
0 );
CheckPassed(rval);
test the synchronous, multi-call apIs.
//
// Now test the synchronous, non-one-call APIs.
//
rval = Tss2_Sys_GetTestResult_Prepare( systemContext );
CheckPassed(rval);
// Execute the command synchronously.
rval = Tss2_Sys_Execute( systemContext );
CheckPassed(rval);
// Get the command results
rval = Tss2_Sys_GetTestResult_Complete( systemContext, &outData,
&testResult );
CheckPassed(rval);
test the asynchronous, multi-call apIs.
//
// Now test the asynchronous, non-one-call interface.
//
rval = Tss2_Sys_GetTestResult_Prepare( systemContext );
CheckPassed(rval);Chapter 7 ■ tpM Software StaCk
93
// Execute the command asynchronously.
rval = Tss2_Sys_ExecuteAsync( systemContext );
CheckPassed(rval);
// Get the command response. Wait a maximum of 20ms
// for response.
rval = Tss2_Sys_ExecuteFinish( systemContext, 20 );
CheckPassed(rval);
// Get the command results
rval = Tss2_Sys_GetTestResult_Complete( systemContext, &outData,
&testResult );
CheckPassed(rval);
// Tear down the system context.
TeardownSysContext( systemContext );
}
```

### SAPI测试代码
前面出现的GetTestResult作为一个测试被包含在SAPI测试代码中。这一小节简单介绍一下测试代码的结构和特点。

测试代码中包含了很多SAPI功能测试。但是需要知道的是它绝对不是一个完整的测试。一个完整的测试有许多情况，一个开发者没有时间去写所有的测试。这个测试就是用于最基本功能的检查，有些情况下可能会针对要测试的功能做细分测试。

测试代码在Test\tpmclient子目录下。tpmclient.cpp这个文件包含测试应用的初始化以及主函数（注：现在已改名为tpmclient.int.c）。tpmclient这个子目录下还包含了测试辅助代码。simDriver包含了用于和TPM模拟器通信的设备驱动。resourceMgr子目录包含了RM的示例代码。sample主要包含了应用层的代码，它主要有一下功能：维护会话状态信息，计算HMAC，密码学运算。（注：现在simDriver/resourceMgr/sample目录已经不存在了，参考tpm2-software）

SAPI测试代码的主要设计原则是使用TPM设备本身来做密码学运算。所以不需要使用外部的密码算法库如OpenSSL。这样做的好处有两点：首先，通过调用TPM的密码学命令可以增加SAPI测试代码的覆盖率。第二，测试应用可以作为一个相对独立的应用，不需要以来外部的软件库。另外其实还有第三个好处：开发者认为这样做很酷！应用软件的开发者可以性SAPI的测试代码开始：可以在测试代码中找到一个你想用的命令作为参考，这将大大加速开发者的软件开发进度。

SAPI的测试代码还使用了TSS软件栈的其他部分来做测试：TCTI，TAB，和RM。因为SAPI使用TCTI向TAB发送命令，所以接下来首先介绍一下TCTI。

## TCTI
尽管我们已经介绍了SAPI的函数，但是仍需要解决的疑问是，命令的数据流究竟是怎样被发送到TPM设备的，应用程序又是如何从TPM设备接收命令响应数据的。答案就是TPM命令传输接口（TCTI）。在Tss2_Sys_Initialize的介绍中简单提到过这个问题。Tss2_Sys_Initialize接收一个TCTI上下文结构体作为它的第一个参数。现在我们将详细介绍TCTI。

TCTI上下文结构体用于指示SAPI函数如何与TPM设备通信。这个结构体包含了TCTI最重要的两个函数的指针，transmit和receive；以及相对较少使用的cancel，setLocality，和其他函数的指针。如果一个应用程序需要和多个TPM通信，它可以创建多个TCTI上下文，然后设置好相应与TPM通信的函数指针。

TCTI上下文结构是进程和TPM相关的数据结构，初始化代码负责配置这个数据结构。具体来说，它可以在编译时初始化，也在在OS启动时动态初始化。系统的一些进程必须可以发现TPM设备或者是提前知道远程TPM的相关信息，然后用相应用于通信的函数指针来初始化这个结构。发现设备和初始化的过程已经超出了SAPI和TCTI规范的范畴。

最常用也是必须有的两个函数指针是transmit和receive，它们完成你期望的事情，发送和接受数据。这两个函数都接收一个缓冲区指针和大小作为参数。当SAPI函数准备好发送和接受时就会调用相应的函数，它们会正确地完成该做的任务。

cancel函数用于支持TPM2.0的新功能：在命令发送到TPM之后可以取消命令的执行。这个功能可以让一些耗时的命令中途被取消。比如说，在一些TPM设备上生成秘钥有可能花费超过90秒的时间。如果在此期间系统因为响应用户操作需要进入睡眠，系统在执行睡眠操作之前可以通过这个命令取消TPM操作，从而使系统合理的处理当前的状态然后进入睡眠。

getPollHandles这个函数指针用于SAPI使用异步方式发送和接收命令数据时，也就是执行Tss2_Sys_ExecuteAsync和Tss2_Sys_ExecuteFinish时。这个函数的实现与系统相关，它可以用于等待接收命令响应时机。

最后一个要介绍的函数指针是finalize，它用于在TCTI连接中断之前做一些清理工作。

TCTI可以应用于TPM软件栈中任何标准化的数据流发送和接收的地方。当前的想法是，它可以出现在两个地方：在SAPI和TAB之间，或者RM和设备驱动之间。

## TPM访问代理（TPM Access Broker）
TAB用于多个进程共享一个TPM时的控制和同步操作。当一个进程在发送和接收数据时，其他的进程不能访问TPM。这是TAB主要的任务。TAB的另外一个任务是阻止进程访问不属于他的TPM会话，对象，以及哈希和事件的序列。资源的所有权在使用对应的TCTI连接加载对象，开启会话，或者启动一个事件序列的时候就确定了。

在大多数的实现中，TAB和RM是集成到一起组成一个软件模块。这样做的主要原因是，对RM做一些简单修改就可以完成一个典型的TAB实现。

## 资源管理器
RM的角色和操作系统中的虚拟内存管理器类似。因为TPM通常有非常有限的片上内存资源，所以对象，会话，和操作序列等资源必须换入换出TPM来保证命令的执行。一个TPM命令最多可以使用三个资源实体handle和三个会话handle。所有这些handle都需要在TPM内部的内存中，这样TPM命令才能够执行。RM的工作就是解析TPM命令数据流，决定哪些资源需要加载到TPM中，加载资源之前还要换出一些TPM资源为加载留出足够空间，然后加载所需的TPM资源。对于TPM对象和操作序列来说，因为它们在加载到TPM知道可能包含多个handle，RM需要将这些handle虚拟化之后再返回给调用者。（正是由于这个原因，授权信息的计算不包含handle信息，因为应用看到的是经过RM虚拟化的handle，而TPM则使用实际的handle，这样就会导致授权认证失败。）第18章中还会有更多的详细介绍，现在关于这个话题我们就先介绍到这里。

RM和TAB通常组成一个软件组件，就是TAB/RM，并且一个TPM对应一个这样的软件组件；这是一个软件设计上的决定，但通常也是这样实现的。但是如果想要一个TAB/RM提供对多个TPM的访问功能，那么TAB/RM就需要跟踪所有的handle的信息，比如handle属于哪一个TPM，并且将它们分开管理。这种情况已经超出了TSS规范的范畴。所以不管是使用不同的可执行文件还是使用同一份代码中的不同列表来实现不同TPM的资源界限，这一层必须要实现不同TPM资源的清晰区分。

TAB和RM在大部分情况下对上层的软件栈来说是透明的，但是同时这两层是可选的。对于上层软件栈来说，不管是直接和TPM设备通信还是通过TAB/RM来发送和接收TPM命令数据，它们的操作都是一样的，这也就是“透明”的含义。但是，如果没有TAB/RM这一层，上层的软件必须在发送TPM命令之前，实现TAB/RM的操作，这样就能保证命令正确执行。通常情况下，应用程序会在多线程或者多进程环境中实现一个TAB/RM来隔离底层的信息。单线程或者高度嵌入式的应用通常不需要TAB/RM层。

## 设备驱动
当FAPI，ESAPI，SAPI，TCTI，TAB和RM完成它们的任务之后，就到最后一级的设备驱动程序登场了。设备驱动程序接收命令数据缓冲区及其大小，然后做必要的硬件操作来想TPM设备发送数据。当收到上层软件的请求之后，设备驱动会发送数据并阻塞直到命令响应数据准备好，然后返回到上层软件。

设备驱动用于和TPM通信的物理层和逻辑层接口不在TPM2.0规范的范畴内，它们在平台相关的规范中定义。目前为止，在PC上可选的TPM接口是FIFO和CRB(Command Response Buffer)。FIFO是先进先出字节传输接口，它使用固定的地址来发送和接收数据，同时还附带一些用于握手和状态操作的地址（寄存器接口）。FIFO接口在TPM2.0中没有太大的变化，只有一小部分的修改。FIFO接口可以使用SPI或者LPC总线。

## 总结
现在为止我们完成了TSS所有软件层的讨论，这些标准的API层用于“驱动”TPM设备工作。你可以根据具体的需求使用不同的层。后续的章节中会用到这些软件层，尤其是FAPI和SAPI层。所以在学习后续的代码示例时可以参考这一章的内容。

