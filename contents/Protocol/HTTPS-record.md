# HTTPS 温故知新（二） —— TLS 记录层协议


<p align='center'>
<img src='https://img.halfrost.com/Blog/ArticleImage/119_0.png'>
</p>

TLS 记录协议是一个层次化的协议。在每一层中，消息都可能包含长度、描述、内容等字段。记录协议主要功能包括装载了被发送的消息，将数据分片为可管理的块，有选择地压缩数据，应用 MAC，加密，传输最终数据。接收到的数据被解密，验证，解压缩，重组，然后传递给高层应用。

特别需要注意的是一条记录消息的类型 type 和长度 length 是不会被加密保护的，它们是明文的。如果这个信息本身是敏感的，应用设计者可能会希望采取一些措施(填充 padding，覆盖流量cover traffic) 来以减小信息泄露。

这篇文章我们重点来讨论一下 TLS 1.2 和 TLS 1.3 在记录层上的不同。先从共同点开始。

## 一、TLS 记录层的连接状态

一个 TLS 连接的状态就是 TLS 记录协议的操作环境。它指定了一个压缩算法，一个加密算法，一个 MAC 算法。此外，这些算法的参数必须是已知的：用于 connection 连接的读、写两个方向的 MAC 密钥和块加密密钥。逻辑上总是有4个状态比较突出：可读和可写状态，挂起的读和写状态。所有的记录协议都在可读写状态下处理。挂起状态的安全参数可以通过 TLS 握手协议来设置，而 ChangeCipherSpec 可以有选择地设置当前状态为挂起状态，在这种情况下适当的当前状态被设置，并被挂起状态所替代; 挂起状态随后会被重新初始化为一个空状态。将一个状态未经安全参数的初始化就设置为一个当前状态是非法的。初始当前状态一直会指定不使用加密，压缩或 MAC。

> ChangeCipherSpec 在官方 TLS 1.3 的规范中已经去掉了，但是为了兼容老的 TLS 1.2 或者网络中间件，这个协议还可能存在。

简单来说，Client 和 Server 在建立链接之前都处于：

```c
      pending read status 待读状态
      pending write status 待写状态
```

一旦接收到对端的 ChangeCipherSpec 消息以后，Client 和 Server 就会开始转换为：

```c
      current read status 可读状态
      current write status 可写状态
```

在收到对端的 ChangeCipherSpec 之前，所有的 TLS 握手消息都是明文处理的，没有安全性和完整性的保护。一旦所有的加密参数都准备好，就会转换成可读可写状态，进入到了可读可写状态以后就会开始加密和完整性保护了。

一个 TLS 连接读写状态的安全参数可以通过提供如下值来设定：

```c
      enum { server, client } ConnectionEnd;
```

- 连接终端:    
  在这个连接中这个实体被认为是 "client" 或 "server"。
  
```c
      enum { tls_prf_sha256 } PRFAlgorithm;
```

- PRF 算法:    
  被用于从主密钥生成密钥的算法。**在 TLS 1.2 中 PRF 默认使用加密基元是 SHA256 算法**。在 TLS 1.2 握手协议中，需要通过该函数将预备主密钥转换为主密钥，主密钥转换为密钥块。**在 TLS 1.3 中这块发生了很大的变化，具体的变化在握手协议中再讨论**。

```c
      enum { null, rc4, 3des, aes }
        BulkCipherAlgorithm;
        
      enum { stream, block, aead } CipherType;
```

- 块加密算法：    
  被用于块加密的算法。它包含了这种算法的密钥长度，它是成块加密，流加密，或 AEAD 加密，密文的块大小(如果合适的话)，和显示和隐式初始化向量(或 nonces)的长度。


```c
      enum { null, hmac_md5, hmac_sha1, hmac_sha256,
           hmac_sha384, hmac_sha512} MACAlgorithm;
```

- MAC 算法:  
  被用于消息验证的算法。包含了 MAC 算法返回值的长度。


```c
      enum { null(0), (255) } CompressionMethod;
      
      /* CompressionMethod, PRFAlgorithm,
         BulkCipherAlgorithm, 和 MACAlgorithm 指定的算法可以增加 */   
```

- 压缩算法:  
  用于数据压缩的算法。被规范必须包含算法执行压缩所需的所有信息。

- 主密钥:    
  在连接的两端之间共享的 48 字节密钥。
  
- 客户端随机数:  
  由客户端提供的 32 字节随机数。
               
- 服务器随机数:  
  由服务器提供的 32 字节随机数。

根据上面这些参数，我们得到，安全参数的数据结构，如下：

```c
      struct {
          ConnectionEnd          entity;
          PRFAlgorithm           prf_algorithm;
          BulkCipherAlgorithm    bulk_cipher_algorithm;
          CipherType             cipher_type;
          uint8                  enc_key_length;
          uint8                  block_length;
          uint8                  fixed_iv_length;
          uint8                  record_iv_length;
          MACAlgorithm           mac_algorithm;  /*mac 算法*/
          uint8                  mac_length;     /*mac 值的长度*/
          uint8                  mac_key_length; /*mac 算法密钥的长度*/
          CompressionMethod      compression_algorithm;
          opaque                 master_secret[48];
          opaque                 client_random[32];
          opaque                 server_random[32];
      } SecurityParameters;
```

TLS 握手协议会填充好上述的加密参数，然后 TLS 记录层会使用安全参数产生如下的 6 个条目(其中的一些并不是所有算法都需要的，因此会留空):

```c
      client write MAC key
      server write MAC key
      client write encryption key
      server write encryption key
      client write IV
      server write IV
```

这里是 2 套 MAC 密钥，加密密钥和初始化向量。原因是因为 Client 和 Server 通信的双方分别维护着自己的安全参数 SecurityParameters。

当 Server 接收并处理记录时会使用 Client 写参数，反之亦然。例如：Client 使用 client write MAC key、client write encryption key、client write IV 密钥块加密消息，Server 接收到以后，也需要使用 Client 的 client write MAC key、client write encryption key、client write IV 的密钥快进行解密。


一旦安全参数被设定且密钥被生成，连接状态就可以将它们设置为当前状态来进行初始化。这些当前状态必须在处理每一条记录后更新。每个连接状态包含如下元素：

- 压缩状态：  
  压缩算法的当前状态。**一般不启用压缩**，压缩可能会导致安全问题，具体问题在 TLS 安全的那篇文章里面再仔细分析。

- 密钥状态：  
  加密算法的当前状态，即每个连接使用的加密算法和加密算法使用的密钥块。这个状态由连接的预定密钥组成。对于流密码，这个状态也将包含对流数据进行加解密所需的任何必要的状态信息。
        
- MAC 密钥：  
  当前连接的 MAC 密钥。

- 序列号：  
  每个连接状态包含一个序列号，读状态和写状态分别维持一个序列号。当一个连接的状态被激活时序列号必须设置为 0。序列号的类型是 uint64，所以序列号大小不会超过 2^64-1。序列号不能 warp。如果一个 TLS 实现需要 warp 序列号，则必须重新协商。一个序列号在每条记录信息被发送之后自动增加。特别地，在一个特殊连接状态下发送的第一条记录消息必须使用序列号 0。**序列号本身是不包含在 TLS 记录层协议消息中的**。


## 二. TLS 记录层协议的处理步骤

TLS 记录层协议处理上层传来的消息，处理步骤主要分为 4 步：

<p align='center'>
<img src='https://img.halfrost.com/Blog/ArticleImage/119_1.png'>
</p>

- 数据分块
- 数据压缩
- 加密和完整性保护(在 TLS 1.2 中主要分三种模式：流加密模式、分组模式、AEAD 模式，在 TLS 1.3 中**只有** AEAD 模式)
- 添加消息头

接下来依次来看看这 4 步处理步骤的细节：

### 1. 数据分块

#### (1) TLS 1.2

记录层将信息块分段为以 2^14 字节或更小的块存储数据的 TLSPlaintext。**TLSPlaintext 就是 TLS 记录层分块后的数据结构**。Client 信息边界并不在记录层保留(即，多个同一内容类型的 Client 信息会被合并成一个 TLSPlaintext，或者一个消息会被分片为多个记录)。

```c
      struct {
          uint8 major;
          uint8 minor;
      } ProtocolVersion;

      enum {
          change_cipher_spec(20), 
          alert(21), 
          handshake(22),
          application_data(23), 
          (255)
      } ContentType;

      struct {
          ContentType type;
          ProtocolVersion version;
          uint16 length;
          opaque fragment[TLSPlaintext.length];
      } TLSPlaintext;
```

- type:    
  用于处理封装的分片的高层协议类型。

- version:  
  协议的版本。TLS 1.2 的版本是{3,3}。版本值 3.3 是基于历史的，因为 TLS 1.0 使用的是{3,1}。需要注意的是一个支持多个版本 TLS 的 Client 在收到 ServerHello 之前可能并不知道最终版本是什么。

- length:  
  TLSPlaintext.fragment 的长度(以字节计)。这个长度不能超过 2^14.

- fragment:  
  应用数据。这种数据是透明的并且作为一个独立的块由 type 域所指定的高层协议来处理。

实现上不能发送 fragments 长度为 0 的握手，alert 警报，或 ChangeCipherSpec 内容类型。发送 fragment 长度为 0 的应用数据在进行流量分析时是有用的。

注意：不同 TLS 记录层内容类型的数据可能是交错的。应用数据的传输优先级通常低于其它内容类型。然而, 记录必须以记录层能提供保护的顺序传递到网络中。接收者必须接收并处理一条连接中在第一个握手报文之后交错的应用层流量。

相同协议的多个子消息，是可以合并到一个 TLS 记录层协议的数据结构中，例如握手协议中的多个子消息，type 都是 handshake，只不过是数据段长度不同，但是数据结构都可以是 TLSPlaintext。


#### (2) TLS 1.3

在 TLS 1.2 规范中，高层协议有 4 个，分别是 change\_cipher\_spec、alert、handshake、application\_data。在 TLS 1.3 规范中高层协议也有 4 个，分别是 alert、handshake、application\_data，heartbeat。

```c
      struct {
          ContentType type;
          ProtocolVersion legacy_record_version;
          uint16 length;
          opaque fragment[TLSPlaintext.length];
      } TLSPlaintext;
```

TLSPlaintext 数据结构在 TLS 1.3 中没有发生变化，字段的含义也是完全一致的。不过字段的值新增了几个。

ContentType 在 TLS 1.3 中新增加了 heartbeat(24) 类型。

```c
      enum {
          invalid(0),
          change_cipher_spec(20),
          alert(21),
          handshake(22),
          application_data(23),
          heartbeat(24),  /* RFC 6520 */
          (255)
      } ContentType;
```

ProtocolVersion 是为了兼容 TLS 1.3 之前的版本，该字段在 TLS 1.3 中已经被废弃了。

在 TLS 1.3 中，version 为 0x0304，过去版本与 version 的对应关系如下：

|协议版本|version|
|:---:|:---:|
|TLS 1.3 |0x0304 |
|TLS 1.2 |0x0303 |
|TLS 1.1 |0x0302 |
|TLS 1.0 |0x0301 |
|SSL 3.0 |0x0300 |


TLS 1.3 中，相同协议的多个子消息也可以合并成单个 TLSPlaintext，不过 TLS 1.3 中的规则比 TLS 1.2 中强制执行的规则更加严格。例如：握手消息可以合并为单个 TLSPlaintext 记录，或者在几个记录中分段，前提是：

- 握手消息不得与其他记录类型交错。也就是说，如果握手消息被分成两个或多个记录，则它们之间不能有任何其他记录。

- 握手消息绝不能跨越密钥更改。实现方必须验证密钥更改之前的所有消息是否与记录边界对齐; 如果没有，那么他们必须用 "unexpected_message" alert 消息终止连接。因为 ClientHello，EndOfEarlyData，ServerHello，Finished 和 KeyUpdate 消息可以在密钥更改之前立即发生，所以实现方必须将这些消息与记录边界对齐。

另外在 TLS 1.3 中 Alert 消息禁止在记录之间进行分段，并且多条 alert 消息不得合并为单个 TLSPlaintext 记录。换句话说，具有 alert 类型的记录必须只包含一条消息。

以上是 TLS 1.3 和 TLS 1.2 在 TLS 记录层数据分块上的不同，TLS 1.2 有的一般特性，在 TLS 1.3 中也同样存在，例如：实现方绝不能发送握手类型的零长度片段，即使这些片段包含填充；应用数据片段可以拆分为多个记录，也可以合并为一个记录。以下 TLS 1.3 和 TLS 1.2 相同点就不在赘述了，只会对比出 TLS 1.3 和 TLS 1.2 的不同点。


### 2. 数据压缩

#### (1) TLS 1.2

所有的记录都要利用在当前会话状态中定义的压缩算法进行压缩。这里的压缩算法必须一直是激活的；然而，初始时它被定义为 CompressionMethod.null。压缩算法将一个 TLSPlaintext 结构转换为一个 TLSCompressed 结构，在连接状态被激活时压缩函数会由默认状态信息进行初始化。具体的可以参考 [RFC3749](https://tools.ietf.org/html/rfc3749) 中描述的用于 TLS 的压缩算法。

压缩必须是无损的，也不能增加内容的长度超过 1024 字节。如果解压函数遇到一个 TLSCompressed.fragment，其解压后的函数超过 2^14 字节，则必须报告一个 fatal 压缩失败错误。

结果压缩以后，消息结构如下：

```c
      struct {
          ContentType type;       /* same as TLSPlaintext.type */
          ProtocolVersion version;/* same as TLSPlaintext.version */
          uint16 length;
          opaque fragment[TLSCompressed.length];
      } TLSCompressed;
```

- length:  
  TLSCompressed.fragment 的长度(以字节计算)。这个长度不能超过 2^14 + 1024。

- fragment:  
  TLSPlaintext.fragment 压缩后的形态。

注意：一个 CompressionMethod.null 的操作是恒等操作，不改变任何域。即，如果不压缩，TLSPlaintext 记录和 TLSCompressed 记录是等价的。

另外，解压函数还必须要保证消息不会导致内部缓存溢出。

**由于安全问题，在 TLS 1.2 中，压缩算法一般不启用**。


#### (2) TLS 1.3

在 TLS 1.3 中完全删除了这一块部分，消息验证码都是由 AEAD 承担。


### 3. 加密和完整性保护

数据经过压缩以后(如果有压缩的话)，接下来就该进行加密和完整性保护了。在 TLS 1.2 中是通过加密和 MAC 值计算，把 TLSCompressed 转换成 TLSCiphertext 。在 TLS 1.3 中是把 TLSInnerPlaintext 转换成 TLSCiphertext。

#### (1) TLS 1.2

先来看看 TLS 1.2 中记录协议是如何进行加密和完整性保护的。在 TLS 1.2 中，记录层协议有 3 种加密方式：

```c
      struct {
          ContentType type;
          ProtocolVersion version;
          uint16 length;
          select (SecurityParameters.cipher_type) {
              case stream: GenericStreamCipher;
              case block:  GenericBlockCipher;
              case aead:   GenericAEADCipher;
          } fragment;
      } TLSCiphertext;
```


- type:  
  这里的 type 值与 TLSCompressed.type 相同。

- version:  
  这里的 version 值与 TLSCompressed.version 相同。

- length:  
  length 代表接下来的 TLSCiphertext.fragment 的长度(以字节为单位)。这个长度不能超过2^14 + 2048。

- fragment:  
  TLSCompressed.fragment 的加密形态, 加密以后末尾要加上MAC。
  
这里要说明一下末尾加上的 MAC 值包含哪些内容。**记录的 MAC 包含了一个序列号，这个序列号用于感知丢失、增加和重复的消息**。


#### I. 标准流加密或者空加密

流加密(包括 BulkCipherAlgorithm.null)将 TLSCompressed.fragment 结构转换为流的 TLSCiphertext.fragment 结构。

```c
        stream-ciphered struct {
          opaque content[TLSCompressed.length];
          opaque MAC[SecurityParameters.mac_length];
      } GenericStreamCipher;
```

MAC 值是通过如下方式产生的：

```c
        MAC(MAC_write_key, seq_num +
                            TLSCompressed.type +
                            TLSCompressed.version +
                            TLSCompressed.length +
                            TLSCompressed.fragment);
```

上面的“+”表示连接。


- seq\_num:  
  这个记录的序列号。

- MAC:  
  由 SecurityParameters.mac\_algorithm 指定的 MAC 算法。

需要注意的是 MAC 值是在加密之前计算出来的。流加密算法加密整个块，包括 MAC。对于不使用同步向量的流加密算法(如 RC4)，流加密算法状态在一个记录的末尾就可以简单地用于随后的包。如果密码算法族是 TLS\_NULL\_WITH\_NULL\_NULL，加密则由同一性操作构成(即数据不加密，MAC 大小是0，意味着不使用 MAC)。对于空加密和流加密，TLSCiphertext.length 等于 TLSCompressed.length 加上 SecurityParameters.mac\_length。即 MAC 的长度由加密参数 SecurityParameters 来决定的。

流加密的完整流程画出来如下图：


<p align='center'>
<img src='https://img.halfrost.com/Blog/ArticleImage/119_2.png'>
</p>

流密码或者空加密模式下，先计算 MAC 值，MAC 值是有 5 个输入参数，序列号、type 类型，protocol version 协议版本、fragment length 长度和 fragment 。计算出 MAC 值以后，再通过加密算法加密，生成最终的密文。这里采用的就是 MAC-then-encrypt 模式。(**注意：消息头是不加密的**)

计算 MAC 值的时候，入参里面有 seq num 序列号。**序列号的主要目的是为了防止重放攻击**。客户端会在内存中记录 client\_send 和 client\_recv。客户端每发送一条消息，client\_send 会加一，每接收一条服务端发来的消息，client\_recv 会加一。服务端也会在内存中记录 server\_send 和 server\_recv，作用和客户端的作用一样。服务端每发送一条消息，server\_send 会加一，每接收一条客户端发来的消息，server\_recv 会加一。如果发送和接收都正常，那么 client\_send = server\_recv、client\_recv = server\_send。client\_send、server\_recv、client\_recv、server\_send 默认值都是 0 。

序列号和 MAC 值的关系是，本次发送或者接收的消息的实际序列号比计算 MAC 值的序列号多一。为什么这么说呢？举个例子：在发送第 5 条的消息，消息到了 TLS record 层，由于本次还没有完全发送成功，所以当前 client\_send = 4，MAC 值计算的时候，序列号就是 4，当发送成功以后，client\_send ++ 等于 5 。同理，在接收第 9 条消息的时候，当前 client\_recv = 8，在验证 MAC 的时候，也是取当前 client\_recv 的值进行验证，当确认这条消息无误以后，client\_recv ++ 等于 9 。

#### II. 分组加密

在 TLS 1.2 中，分组加密主要是指的 CBC 块加密。

对于块加密算法(如 3DES 或 AES)，加密和 MAC 函数将 TLSCompressed.fragment 转换为 TLSCiphertext.fragment 结构块。

```c
   struct {
          opaque IV[SecurityParameters.record_iv_length];
          block-ciphered struct {
              opaque content[TLSCompressed.length];
              opaque MAC[SecurityParameters.mac_length];
              uint8 padding[GenericBlockCipher.padding_length];
              uint8 padding_length;
          };
      } GenericBlockCipher;
```

MAC 的生成方法流加密中的 MAC 生成方式相同。

```c
        MAC(MAC_write_key, seq_num +
                            TLSCompressed.type +
                            TLSCompressed.version +
                            TLSCompressed.length +
                            TLSCompressed.fragment);
```

- IV:  
  初始化向量(IV)应该随机产生，并且必须是不能预测的。需要注意的是在 TLS 1.1 以前的版本是没有 IV 域的。以前的记录中最后一个密文块(CBC 分组最后一组的剩余)被用作 IV。使用随机的 IV 向量是为了阻止在 [[CBCATT]](https://tools.ietf.org/html/rfc5246#ref-CBCATT) 中描述的攻击。对于块加密，IV 的长度是 SecurityParameters.record\_iv\_length 的值，这个值等于 SecurityParameters.block\_size。

- padding:  
  padding 用于强制使明文的长度是块加密块长度的整数倍，它可能是任意长度，最长是 255 字节，只要它能让 TLSCiphertext.length 是块长度的整数倍。可能需要长于所需的长度来阻止对基于对交换消息的长度的分析的协议的攻击。在 padding 数据向量中的每个 uint8 必须用 padding 的长度值填充。接收者必须检查这个 padding 且必须使用 bad\_record\_mac alert 警告消息来暗示 padding 错误。

- padding\_length:  
  padding 的长度必须使 GenericBlockCipher 的总长度是密码块长度的整数倍。合法的取值范围是从 0 到 255，包含 0 和 255。这个长度指定了 padding 字段的长度但不包含 padding\_length 字段的长度。

密文数据的长度(TLSCiphertext.length)大于 SecurityParameters.block\_length, TLSCompressed.length, SecurityParameters.mac\_length, 和 padding\_length 之和。
例如: 如果块长度是 8 字节，内容长度(TLSCompressed.length)是 61 字节，MAC 长度是 20 字节，则填充 padding 之前的长度是 81 字节(不包含 IV)。为了使总长度是 8 字节(块长度)的偶数倍，填充长度模 8 必须等于 7。即填充长度可以是 7, 15, 23, 以此类推, 直到 255。如果有必要使填充长度最小，为 7，则填充必须是 7 字节，由于填充的最后一个字节表示填充的长度，所以实际上填充的每个字节都应该为 6。因此，GenericBlockCipher 在块加密前的最后 8 个字节可能是 xx 06 06 06 06 06 06 06, 这里 xx 是 MAC 的最后一个字节。最后一个 06 就是 padding\_length，它代表了 padding 字段的长度，即 6 个字节。

```c
        TLSCiphertext.length >= SecurityParameters.block_length 
        					+ TLSCompressed.length
        					+ SecurityParameters.mac_length
        					+ padding_length
```

注意：对于 CBC 模式(密文分组链接模式)的块加密，关键的是记录的整个明文在传输任何密文之前就已被知道。否则，攻击者可能会发动 [[CBCATT]](https://tools.ietf.org/html/rfc5246#ref-CBCATT) 中描述的攻击。[[CBCTIME]](https://tools.ietf.org/html/rfc5246#ref-CBCTIME) 阐述了一个基于 MAC 计算时间的针对 CBC 填充的定时攻击。为了防御此类攻击，无论填充是否正确，实现方都必须确保记录处理时间基本相同。通常，做到这一点最好的方式是即使填充不正确也计算 MAC，然后才拒绝该数据包。例如，如果填充看起来不正确，实现方可能假设零长度填充然后计算 MAC。这留下了一个小的定时信道，因为 MAC 计算性能在某种程度上取决于数据分片的长度，但不能确信这个长度会大到足够被利用，这是因为现存 MAC 的块大而定时信号的长度小。


分组加密的完整流程画出来如下图：


<p align='center'>
<img src='https://img.halfrost.com/Blog/ArticleImage/119_3.png'>
</p>

相比较流加密，分组加密多了 padding 和 IV。

#### III. AEAD 模式

AEAD 加密相比前两种加密方式，使用更加简单，安全性更高。因为它不需要使用者考虑 HMAC 算法，并且也不需要初始化向量与填充 padding。

AEAD 密码套件主要有 3 种：

|AEAD 模式 | 加密 | 密码套件|
|:----:|:-----:|:-----:|
|GCM|AES-128-GCM|TLS\_DHE\_RSA\_WITH\_AES\_128\_GCM\_SHA256|
|CCM|AES-128-CCM|TLS\_RSA\_WITH\_AES\_128\_CCM|
|ChaCha20-Poly1305|ChaCha20-Poly1305|ECDHE-ECDSA-CHACHA20-POLY1305|

在 TLS 协议中，CCM 用的比较少，GCM 用的比较多，尤其用在具有 AES 加速的 CPU 上。ChaCha20-Poly1305 是 Google 发明的由 ChaCha20 流加密，Poly1305 消息认证码组合的一种加密算法，用在移动端上比较多。

对于 AEAD [[AEAD]](https://tools.ietf.org/html/rfc5246#ref-AEAD) 加密(如：[[CCM]](https://tools.ietf.org/html/rfc5246#ref-CCM) 或 [[GCM]](https://tools.ietf.org/html/rfc5246#ref-GCM)), AEAD 函数将 TLSCompressed.fragment 结构转换为 AEAD TLSCiphertext.fragment结构。

```c
        struct {
         opaque nonce_explicit[SecurityParameters.record_iv_length];
         aead-ciphered struct {
             opaque content[TLSCompressed.length];
         };
      } GenericAEADCipher;
```

AEAD 加密的输入有：单个密钥，一个 nonce，一块明文(就是 TLSCompressed.fragment)，和被包含在验证检查中的“额外数据”（在 [[AEAD]](https://tools.ietf.org/html/rfc5246#ref-AEAD) 2.1节中描述）。密钥是 client\_write\_key 或者 server\_write\_key。不使用 MAC 密钥。

每个 AEAD 密码族必须指定提供给 AEAD 操作的 nonce 是如何构建的，GenericAEADCipher.nonce\_explicit 部分的长度是什么。在很多情况下，使用在 [[AEAD]](https://tools.ietf.org/html/rfc5246#ref-AEAD) 3.2.1节中描述的部分隐藏的 nonce 技术是合适的；record\_iv\_length 就是 GenericAEADCipher.nonce\_explicit 的长度。在这种情况下，隐式部分应该作为 client\_write\_iv 和 server\_write\_iv 从 key\_block 中(在6.3节中描述)推导出来，显示部分被包含在 GenericAEAEDCipher.nonce\_explicit 中。

明文是 TLSCompressed.fragment。

额外的验证数据(我们表示为 additional\_data)定义如下：

```c
        additional_data = seq_num + TLSCompressed.type +
                        TLSCompressed.version + TLSCompressed.length;
```

这里“+”表示连接。

AEAD 的输出由 AEAD 加密操作所产生的密文输出构成。长度通常大于 TLSCompressed.length，在量上会随着 AEAD 加密的不同而不同。因为加密可能包含填充，开销的大小可能会因 TLSCompressed.length 值而不同。每种 AEAD 加密不能产生大于 1024 字节的长度。

```c
        AEADEncrypted = AEAD-Encrypt(write_key, nonce, plaintext,
                                   additional_data)
```

为了解密和验证，加密算法将密钥、nonce、“额外数据”和 AEADEncrypted 的值作为输入。输出要么是明文要么是解密失败导致的错误。这里没有分离完整性检查。即：

```c
        TLSCompressed.fragment = AEAD-Decrypt(write_key, nonce,
                                            AEADEncrypted,
                                            additional_data)
```

如果解密失败，会产生一个 bad\_record\_mac alert 消息。

AEAD 加密的完整流程画出来如下图：


<p align='center'>
<img src='https://img.halfrost.com/Blog/ArticleImage/119_4_.png'>
</p>

相比较流加密，AEAD 加密只多了 Nonce。

#### (2) TLS 1.3

在 TLS 1.3 中，只有一种加密和完整性保护的方法了，就是“具有关联数据的认证加密”(AEAD)[[RFC5116]](https://tools.ietf.org/html/rfc5116)。AEAD 功能提供统一的加密和认证操作，将明文转换为经过认证的密文，然后再返回。每个加密记录由一个明文标题后跟一个加密的主体组成，该主体本身包含一个类型和可选的填充。

```c
      struct {
          opaque content[TLSPlaintext.length];
          ContentType type;
          uint8 zeros[length_of_padding];
      } TLSInnerPlaintext;

      struct {
          ContentType opaque_type = application_data; /* 23 */
          ProtocolVersion legacy_record_version = 0x0303; /* TLS v1.2 */
          uint16 length;
          opaque encrypted_record[TLSCiphertext.length];
      } TLSCiphertext;
```

- content:  
	TLSPlaintext.fragment 值，包含握手或警报消息的字节编码，或要发送的应用数据的原始字节。
	
- type:  
	TLSPlaintext.type 值，包含记录的内容类型。

- zeros:  
	在类型字段之后的明文中可以出现任意长度的零值字节。只要总数保持在记录大小限制范围内，这个字段为发件人提供了按所选的量去填充任何 TLS 记录的机会。

- opaque\_type:  
	TLSCiphertext 记录外部的 opaque\_type 字段始终设置为值23(application\_data)，以便与解析以前版本的 TLS 的中间件向外兼容。解密后，在 TLSInnerPlaintext.type 中找到记录的实际内容类型。


- legacy\_record\_version:  
	legacy\_record\_version 字段始终为 0x0303。TLS 1.3 TLSCiphertexts 在协商 TLS 1.3 之后才生成，因此没有历史兼容性问题可能会收到其他值。请注意，握手协议(包括 ClientHello 和 ServerHello 消息)会对协议版本进行身份验证，因此该值是多余的。

- length:  
	TLSCiphertext.encrypted\_record 的长度(以字节为单位)，它是内容和填充的长度之和，加上内部内容类型的长度加上 AEAD 算法添加的任何扩展。长度不得超过 2 ^ 14 + 256 字节。接收超过此长度的记录的端点必须使用 "record\_overflow" alert 消息终止连接。


- encrypted\_record:  
  AEAD 加密形式的序列化 TLSInnerPlaintext 结构。


### 4. 添加消息头

经过加密以后，得到了 TLSCiphertext 数据结构，再添加上消息头后，统一传给 TCP/UDP 层。在 TLS 1.2 和 TLS 1.3 中，添加的消息头都是一样的。

在 TLS 1.2 中，消息头字段有下面这 3 个。

```c
          ContentType type;
          ProtocolVersion version;
          uint16 length;
```

由于为了兼容 TLS 1.3 之前的版本，ProtocolVersion 还是需要保留，但是实际上在 TLS 1.3 的规范中已经不再使用了。在 TLS 1.3 中，消息头字段月有下面这 3 个。


```c
          ContentType type;
          ProtocolVersion legacy_record_version;
          uint16 length;
```


## 三. TLS 记录层协议中的密钥计算

### 1. TLS 1.2 中的密钥计算

TLS 记录协议需要一个算法从握手协议提供的安全参数中生成当前连接状态所需的密钥。

主密钥被扩大为一个安全字节序列，它被分割为一个客户端写 MAC 密钥，一个服务端写 MAC 密钥，一个客户端写加密密钥，一个服务端写加密密钥。它们中的每一个都是从字节序列中以上述顺序生成。未使用的值是空。一些 AEAD 加密可能会额外需要一个客户端写 IV 和一个服务端写。

生成密钥和 MAC 密钥时，主密钥被用作一个熵源。

为了生成密钥数据，计算

```c
            key_block = PRF(SecurityParameters.master_secret,
                      "key expansion",
                      SecurityParameters.server_random +
                      SecurityParameters.client_random);
```

>这里用到了 PRF 算法，关于这个算法笔者会单独写一篇文章来对比 TLS 1.2 和 TLS 1.3 在这个算法上的不同。

直到产生足够的输出。然后，key\_block 会按照如下方式分开：

```c
      client_write_MAC_key[SecurityParameters.mac_key_length]
      server_write_MAC_key[SecurityParameters.mac_key_length]
      client_write_key[SecurityParameters.enc_key_length]
      server_write_key[SecurityParameters.enc_key_length]
      client_write_IV[SecurityParameters.fixed_iv_length]
      server_write_IV[SecurityParameters.fixed_iv_length]
```

目前，client\_write\_IV 和 server\_write\_IV 只能由 [[AEAD]](https://tools.ietf.org/html/rfc5246#ref-AEAD) 3.2.1节中描述的隐式 nonce 技术生成。

实现注记：当前定义的密码协议族使用最多的是 AES\_256\_CBC\_SHA256。它需要 2 x 32 字节密钥和 2 x 32 字节 MAC 密钥，它们从 128 字节的密钥数据中产生。


### 2. TLS 1.3 中的密钥计算





------------------------------------------------------

Reference：  

《深入浅出 HTTPS》      
[TLS\_1.3\_Record\_Protocol](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/TLS_1.3_Record_Protocol.md#%E4%B8%80-record-layer)  
[TLS 1.2 The TLS Record Protocol](https://tools.ietf.org/html/rfc5246#page-15)


> GitHub Repo：[Halfrost-Field](HTTPS://github.com/halfrost/Halfrost-Field)
> 
> Follow: [halfrost · GitHub](HTTPS://github.com/halfrost)
>
> Source: []()