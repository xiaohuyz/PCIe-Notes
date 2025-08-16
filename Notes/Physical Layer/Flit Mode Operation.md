# Flit Mode Operation 
在本章中，描述了flit的定义和symbol的位置。根据不同的PCIe Gen，Flit会被使用各种不同的编码方式进行编码。

采用8b/10b编码的flit，与non-flit模式下的8b/10b编码的不同：

- 因为Flit Mode下具有固定的TLP和DLP位置，那么用来标记TLP / DLLP边界的**markers**，比如**STP / SDP / END / EDB**不再使用

采用128b/130b编码的flit，与non-flit模式下128b/130b编码的不同：
- 因为Flit Mode下具有固定的TLP和DLP位置，那么**Framing Tokens**便不再使用

> 注意，在使用8b/10b编码和128b/130b编码时，仅仅是编码前的信息不一样，后面的操作和以前是一样的

> 注意，在此处我们使用DLP来表示Flit中与DL层信息有关的部分，而非使用Non Flit Mode下使用的DLLP，实际上，与TL层一样，Flit中的DLP的格式和Non Flit Mode下DLLP的格式是不一样的。

## 1b/1b Encoding for 64.0 GT/s and higher Data Rates
当PCIe工作于64.0 GT/s或者更高速率下时，它使用Flit Mode。在每条lane上，symbol（8 bits）是基本的传输单位，即和8b/10b编码，以及128b/130b编码不一样，1b/1b编码是直接把原始的symbol信息放到每条lane上了。无论是Data Stream还是Ordered Sets，都使用**PAM4**信号传输方式。PAM4工作于一个**2-bit对齐**的边界上（即**单条lane在一个UI内实际上是传输2 bit数据**）。如图所示，对于一个8bit的symbol，首先传输的是S~1~S~0~，最后传输的是S~7~S~6~。

![image-20240427150238515](C:/Users/hhsjs/AppData/Roaming/Typora/typora-user-images/image-20240427150238515.png)

> UI指的是Time Unit，在Gen1~Gen5中指的是传输1bit数据所消耗的时间，而对于使用PAM4编码的Gen6，指的是传输2bit数据消耗的时间

对于transmit side和receive side的架构图如图所示：

![image-20240427150348010](C:/Users/hhsjs/AppData/Roaming/Typora/typora-user-images/image-20240427150348010.png)

![image-20240427150410934](C:/Users/hhsjs/AppData/Roaming/Typora/typora-user-images/image-20240427150410934.png)

注意，上图的operation并非所有都必须要执行。在Flit级，对于transmit端，首先进行的是CRC计算，而后是FEC生成，这两步是必需的，这2步完成之后，才把TLP / DLP / CRC / FEC一起打包成一个256 Bytes的Flit。而后，将该256 Byte flit分散到每个lane上进行传输，即每个lane分1个byte，之后的操作以lane为基础同步进行。在每个lane上，首先进行scramble（**在需要进行scramble的情况下**） ，而后进行**2 bit边界的Gray Coding**，而后**在需要的情况下**，以2 bit对齐的边界进行precoding。如图，实际分以下几种情况：

1. 需要进行scrambling，在scrambling之后，必须进行Gray Coding。而后执行Precoding
2. 需要进行scrambling，在scrambling之后，必须进行Gray Coding。而后不执行Precoding（TS0 OS或者Precoding disabled）
3. 不进行scrambling，则scrambling / gray coding / precoding这三个步骤都无需进行

因此，实际的格雷编码和precoding过程是针对‘S~1~S~0~'...'S~7~S~6~'这样的pattern的。如图所示，经过scramble的TS1/TS2 OS同样经过precoding。所有的symbols，无论是属于Flit（data block），还是属于ordered set中的**scrambled symbols**，或者是特别提及的pattern（例如**scrambled part** of modified compliance pattern），都必须**经过格雷编码和PAM4编码。**然而，Ordered Set或者其他pattern中的**unscrambled symbols**仅仅经过PAM4编码。

receive side的结构与transmit side类似，仅仅方向相反。首先是PAM4电压被解析成2 bit对齐的数值，而后经过receive side的precoding（如果需要的话），格雷解码，而后**在单bit级别**进行descramble（如果需要）。而后，Data Stream（Flit）将由所有lane的数据聚合而成，而后经过FEC decode and correction，以及CRC校验，最后发送到DL层和TL层。

> FEC的作用是进行轻量化纠错

### PAM4 Signaling
PAM4意为4级脉冲电平幅值调制。这是一种信号调制机制，在该机制下，4 levels（2 bits）**在同一个Unit Interval（UI）下**被编码，从而使得眼图中有3个眼。该信号调制机制适用于64.0 GT/s及以上速率。四个电平采用格雷码00，01，11，10进行编码。而且**采用小端编码**。PAM4编码机制帮助64.0 GT/s速率下的传输保持与32.0 GT/s下传输相同的奈奎斯特频率（因为每个Unit Interval下发送的信息量翻倍，因此无需翻倍传输频率）。采用0，1，2，3来定义这四个电压幅值，其大小为-400mV，-133mV，+133mV和+400mV，而采用的Gray Coding为00b， 01b， 11b， 10b。

> 奈奎斯特频率，即防止信号混叠所需要定义的最小采样频率。
>
> 最大允许的抽样间隔称为奈奎斯特间隔。

如图所示，注意下表中DC-balance Values指的是当设计Ordered Sets，以满足DC Balance时，其对应Field中所需要使用的值。

![image-20240427151832295](C:/Users/hhsjs/AppData/Roaming/Typora/typora-user-images/image-20240427151832295.png)

使用PAM4编码的情况下，**bit error rate（BER）**将会比更低速率下的10^-12^更大。此外，在单条lane上可能会出现burst error（即出现连续的error），且会出现一些lane to lane correlation（即Lane to Lane的相关性会增加，这使得Lane更容易受到RC效应的影响）。因此，必须满足First Bit Error Rate小于10^-6^，这样经过FEC的纠错之后，能满足Flit Error出现的可能性低于3*10^-5^

因此， 采用**Forward Error Correction（FEC）机制**来处理Data Stream中更高的FBER（First Bit Error Rate）。由于**FEC机制只能工作于固定长度的code**，**采用Flit（flow control unit）**来传输Data Stream中的TLP / DLLP。采用轻量化的FEC，以确保低延迟。并且，采用在Flit Mode计算的CRC，来加强FEC工作的可靠性。**在Flit Mode下，link level retry机制工作于flit level**，而非和Non Flit Mode下一样位于DLLP中。在64.0 GT/s及以上速率中，对于Ordered Set，则采用replication来保护（即Ordered Sets中传输的数据实际上是一段一段重复的）。此外，对**经过scramble的bits**，采用precoding机制，来减少burst error中的error数目，这和Gen4 / Gen5中的机制是类似的。

### 1b/1b Scrambling
在64.0 GT/s下的scrambling机制与更低的速率下基本类似。**不同的是针对TS0 / TS1 / TS2 / SKP OS的scramble规则**。**所有的Data Stream bits**和**一些Ordered Set bit**s需要进行scramble

- TS1 and TS2 Ordered Sets：
  - Symbol 0 和 Symbol 8 跳过scrambling
  - Symbol 1-6和Symbol 9-14需要scrambling
  - 如果需要进行**DC Balance**，则Symbol 7和Symbol 15跳过scrambling，如果不需要DC Balance，则这两个symbol需要进行scrambling。
- TS0 Ordered Sets：
  - Symbol 0 和 Symbol 8 跳过scrambling
  - Symbol 1～6和Symbol 9～14使用**NRZ-based scrambling**机制
    - NRZ-based scrambling，意为**首先对所有的8bit进行scramble，而后，对于2i处的bit，不对它进行格雷编码，而是直接让该bit等于2i+1.这就确保了只有voltage level 0和3被发送（因为coding只有可能为00b或者11b）**
    - 对于symbol 7和symbol 15，如果需要进行**DC Balance**，则Symbol 7和Symbol 15跳过scrambling。否则，首先对symbol1-6和symbol9-14进行NRZ-scrambling，而后设置symbol 7和symbol 15，使得symbol1-7，以及symbol9-15满足偶校验
    - TS0中所有的symbols都属NRZ-based的（即对于TS0 OS，它在一个UI内放在lane上的数据要么是00b，要么是11b，永远只有2个电平）。在64.0 GT/s下，也只有TS0 OS被设计为NRZ-based
- SKP Ordered Sets：
  - 处理Data Stream之外的ordered sets和Data Stream之内的ordered sets规则不同，按照各自的规则处理即可
  - 如果receiver发现接收到的Ordered Sets是一个SKP OS，则**对于Block中的任何symbol**，LSFR都不会shift，这一点和更低速率下是一致的。

> 对于NRZ-based scrambling，在2i处的bit，可以直接令其等于2i+1处的值，而跳过gray coding，也可先统一进行gray coding，而后再令其等于2i+1处的值

### Gray Coding at 64.0 GT/s and Higher Data Rates
只有以2 bit为边界，经过scramble的bits需要进行gray coding。在**Data Stream中**的所有bits都经过scramble和gray coding。而在TS1 / TS2 OS中，仅仅有一些symbols经过scramble，正是这些symbols需要gray coding。所有Half-Scrambled的TS0 OS都经历了NRZ-based scrambling，即只有奇数bit经过scrambling和gray coding，而偶数bit强制和奇数bit相等。

格雷编码仅仅在**PAM4模式**下启用（即在64.0 GT/s以及更高速率下）。它在每个lane上，以2bit边界的粒度进行编码。如图所示：

![image-20240427155502097](C:/Users/hhsjs/AppData/Roaming/Typora/typora-user-images/image-20240427155502097.png)

下表展示了在Gray编码下，单级电压的错误对实际编码的影响。如图所示，单极电压的错误最多造成1bit的翻转。

![image-20240427155608795](C:/Users/hhsjs/AppData/Roaming/Typora/typora-user-images/image-20240427155608795.png)

### Precoding at 64.0 GT/s and Higher Data Rates

只有以2bit为边界进行过scramble的bit才会进行precoding。对于**TS0 OS**，**所有symbols**都不经过precoding。
1b/1b下precoding，除了下述规则之外，其余规则与NRZ编码下的precoding规则一致：

- 在Tx侧，**输入P~n~，输出为T~n~**。而T~n-1~是上个UI的precoding结果。注意对于边界处（比如上一个UI没有进行scramble），设置T~n-1~为**00b**。而后T~n~将被转为PAM4编码。
- 注意，在传输过程中，链路的channel可能会注入一个错误E~n~，如果没有错误的话，E~n~就是00b。因此在Rx侧，接收到的是R~n~=T~n~+E~n~
- 实际上，在Rx侧，接收到的是R'~n~=T~n~+e~n~，其中e~n~是channel注入的错误E~n~和内部的DFE propagated error之和。
- 最后Rx侧的输出为P'~n~ 
### Ordered Set Blocks at 64.0 GT/s and Higher Data Rates
在Tx侧和Rx侧，除了SKP OS之外的所有Ordered Set（TS0 / TS1 / TS2 / EIOS / EIEOS / SDS）都是**16 bytes**长度，这一点和128b/130b编码下是一致的。SKP OS在Tx侧是**40 bytes**长，而在Rx侧可能是**24 / 32 / 40 / 48 / 56 bytes**长度（这是因为Rx侧可能会由于Tx和Rx的时钟不同步，而丢弃或者插入SKP OS）。相较于更低速率下的Ordered Sets，64.0 GT/s以及更高速率下的Ordered Set拥有**更大的冗余度**，这是由于PAM4编码下有更大的FBER。

当处理**Data Stream外**的OS时，需要遵循以下规则（有关在什么情况下才算收到了一个对应的Ordered Set）：

- 在**以下条件都满足**的情形下，receiver视为接收到了一个EIEOS
  - 对于一个block，它的**前8bytes**中，有任意5个或以上**对齐**且**连续**的bytes符合EIEOS中的对应bytes
  - 对于一个block，它的**后8bytes**中，有任意5个或以上**对齐**且**连续**的bytes符合EIEOS中的对应bytes
  - block中的**symbol 0或者symbol 8**符合EIEOS中的对应bytes。在这种比较中，receiver允许**忽略最多1 bit的mismatch**。
  - 其实就是经过scramble的前8Bytes / 后8Bytes中，每一半都必须有5个或以上对齐且连续的Byte符合要求，而两个没有经过scramble的byte，有1个符合要求
  
- 在Aligned Phase中，在接收到一个EIEOS时，receiver必须调整它的block边界，且调整幅度必须大于1 UI（即根据收到的EIEOS来调整自己的边界对齐）。**或者**，仅仅当它接收到一个EIEOS，且后面紧跟着一个expected OS的first symbol时，block边界的调整应该小于1UI（这是需要调整边界对齐和不需要调整边界对齐的情况）

- 在搜寻EIEOS时，8 byte边界能够从任意对齐的UI开始。而且在receiver侧，搜索必须在gray coding / precoding / descrambling之前进行

- 在**以下条件都满足**的情形下，receiver视为接收到了一个EIOS
  - 对于一个block，它的**前8bytes**中，有任意5个或以上**对齐**的bytes符合EIOS中的对应bytes，注意此处不要求连续的bytes
  - block中的**symbol 0或者symbol 8**符合EIOS中的对应bytes。当确认收到一个EIOS时，lane便准备进入Electrical Idle。**不关心Block的最后7 bytes的内容**
  
- 在**以下条件都满足**的情形下，receiver视为接收到了一个SKP OS
  - 对于一个block，它的**前8bytes**中，有任意5个或以上**对齐**的bytes符合SKP OS中的对应bytes
  
  - symbol 0是SKP，**或者**symbol8是SKP / SKP_END
  
  - 而后，receiver检查后续的8 bytes对齐的块，而且使用以下规则（**主要是为了判断SKP OS的结束**，因为他的长度是不定的）：
    - 如果在一个8 byte块中，有5个及以上**对齐**的bytes符合SKP_END（这种情况意味着SKP OS的结束），**或者**该块是SKP OS开始后的第5个8 byte块（即byte 40-47），则SKP OS将在**下一个8 bytes对齐的块之后**结束（即**使用SKP_END来结束，或者正好到达SKP OS的最大56 bytes长度后结束**）。

在Data Stream内或者外来infer ordered sets：对于TS0 / TS1 /TS2而言，它们的**大部分bytes**是scrambled的（除了symbol 0和symbol8，这两个symbol从来不经过scramble）。因此，对这三种OS的接收判断，和**在data block之外**EIEOS / EIOS / SKP OS一致，至少要先在symbol 0**或者**symbol 8上match，而后还要match 5个连续byte。
      对于**在data block之内**的SKP / EIOS / EIEOS的判断，因为**它们没有被scramble**，因此使用不同的判别方式。

### Alignment at Block/Flit Level for 1b/1b Encoding
在使用1b/1b编码的情况下，在每条lane上，block**对齐到Ordered Set的边界**。当block对齐完成后，link level的Flit-level对齐便自动出现（因为**Flit的长度是固定的**，且**Flit总是出现于SDS OS + Control_SKP_OS之后**）。当Flit对齐开始时，便开始调整Flit的边界。

Block level的对齐在**Configuration / Recovery states**中，且使用EIEOS来进行。EIEOS有着特定的pattern，用于receiver来判断接收到的bit流中Ordered Set边界的开始和结束。与其他编码方式一样，1b/1b encoding编码下，边界对齐也分三个阶段：

- Unaligned Phase。在一段时间的**Electrical Idle**之后，receiver进入该phase。例如：
  - data rate切换到使用1b/1b编码的data rate
  
  - link退出low-power link state
  
  - 被特定的机制直接导向该phase
  
    在该phase中，receiver检测接收到bit流，来寻找**与EIEOS match的pattern**，注意需要检查全部的128bit，而后按照前述的要求进行匹配。当检测到EIEOS时，根据EIEOS来调整自己的block边界，并且进入Aligned Phase
  
- Aligned Phase
  - 在该phase中，receiver监测bit stream，来探测**EIEOS**和**SDS OS**。
  - 如果LTSSM**处于Recovery.RcvrLock state**，且接收到一个EIEOS bit pattern，其展示的对齐边界和当前对齐边界不一致，则将当前receiver的对齐边界调整到新接收的EIEOS bit pattern。其实这就是刚进入Recovery state后，重新获得Bit Lock和Flit align
  - 如果LTSSM**处于Recovery.RcvrCfg state**,且接收到一个EIEOS bit pattern，其对齐边界和当前对齐边界不一致，且**下一个symbol符合expected Ordered Set的first symbol**，则将当前receiver的对齐调整到新接收的EIEOS bit pattern。TODO，这适用于那种情况？？
  - 如果接收到一个SDS OS，则进入Locked Phase。在接收到SDS OS之后，data stream就算开始了，尽管实际上**Flits在Control SKP OS之后才会开始**。
  
- Locked Phase
  在该phase中，receiver不允许调整它的block或者flit对齐。因为在SDS OS + Control SKP OS的接收之后，会开始接收data blocks。在此时调整block对齐将破坏data blocks的接收。如果data stream**由于Framing Error或者link进入recovery（其实就是link对面的device进入Recovery了，开始发送TS1 OS了）**而结束，则receivers必须返回Unaligned Phase或者Aligned Phase

- 额外的rules：

  - 在Aligned Phase中，当接收到**Control SKP OS**时，receiver必须按照需要调整自己的block alignment
  - 在Locked Phase中，当接收到**Control SKP OS**时，receiver必须按照需要调整自己的block alignment和Flit alignment。TODO，注意看Control SKP OS那一章
  - 在LTSSM切换到Recovery后，Receiver必须丢弃所有接收到的TS OS，直到它接收到一个EIEOS。因为接收到一个EIEOS能够重建block / flit的对齐，从而允许进行TS OS的处理。但是，如果**正是一个EIEOS将receiver的LTSSM从L0切到了Recovery**，那么receiver允许处理该EIEOS后接收到的所有TS，当然也可以无视这个EIEOS，而是等待接收一个新的EIEOS之后，才开始TS OS的接收（因为已经有一个EIEOS了，这一点的处理与Gen3-5类似）
  - 只要Data Stream传输一停止，则允许receiver从locked phase切换到unaligned phase / aligned phase（**切换回这两个phase以方便调整alignment**）
  - 对于Loopback Leads：当LTSSM处于**Loopback.Entry**时，leads必须能够根据接收到的EIEOS来调整自己receiver的Block / Flit alignment。具体方法是，当LTSSM处于**Loopback.Active**时，leads允许发送EIEOS，而后根据looped back bit stream（绕回来的bit流）来调整自己receiver的block alignment
  - 对于Loopback Follower：当LTSSM处于**Loopback.Entry**时，follower必须能够根据接收到的EIEOS来调整自己receiver的block alignment。当LTSSM处于**Loopback.Active**时，follower不允许调整自己receiver的alignment，除非是在安排好的Control SKP OS边界处。因此，当follower开始loop back 接收到的 bit stream时，它会直接被切换到Lock Phase
## Processing of Ordered Sets During Flit Mode Data Stream
注意，本节描述的是Data Stream中的Ordered Sets识别

在Flit Mode下，不使用EDS（End of Data Stream）token来结束数据的传输。一个data stream可以被一个EIOS（在进入**low power state**之前，如L1）结束，或者被一个EIEOS（在进入**recovery**时）结束。在data stream中，SKP OS周期性地出现。当**需要的时候**，EIOS / EIEOS必须在SKP OS的位置传输来代替SKP OS。因此，在设计这3种Ordered Sets时，就设计好即使是经过burst error，这三种Ordered Set也不会互相混淆。

在64.0 GT/s及以上的速率中，按照以下规则来处理Data Stream当中或者结尾的Ordered Set：

- 在OS期待出现的位置，receiver检查Ordered Sets的头部8bytes
- 对于这8 bytes，**任何的5个及以上bytes符合SKP / EIOS / EIEOS**，则认为成功检测到了这一类OS。如果没有达到该要求，则认为是一个Framing Error，link会进入recovery状态（即在Ordered Sets应该出现的位置，却没有检测到Ordered Sets）
- 如果从first 8 bytes中infer到一个SKP OS，则receiver来检查后续的8 bytes对齐的块，来判断SKP OS的结束：
  - 如果在一个8 byte块中，有5个及以上对齐的bytes符合SKP_END，**或者**该块是SKP OS开始后的第5个8 byte块（即byte 40-47），则SKP OS将在**下一个8 bytes对齐的块之后**结束（即**使用SKP_END来结束，或者正好到达SKP OS的最大56 bytes长度后结束**）。
    - 如果通过接收到5个SKP_END来结束SKP OS，则SKP OS结束后，data会恢复传输。
    - 但是如果SKP OS没有通过5个SKP_END来结束，则是一个**Framing Error**（整个SKP OS都传输到最长长度了，但是最后一个8 byte块却不是SKP_END，这明显不对）
- 下列情形均为Framing Error（即使接收到的是一个合适的Ordered Set），而且link必须进入recovery状态：
  - 在任意2个active lanes上面同时接收到了这两种OS的组合：EIEOS和EIOS，EIEOS和SKP OS。即这两种组合不可能同时出现
  - 在所有lane上接收到的SKP OS长度不同
  - 在下列任意情形下，在active lanes上接收到了EIOS和SKP的组合（以下情形是非法的，所谓合法的情况，就是Link进入L0p来升/降lane的情况）：
    - link未启用L0p
    - 接收到EIOS的lanes号是不连续的
    - 接收到SKP OS的lanes号是不连续的
    - 接收到SKP OS的lanes不是一个有效的link宽度（x1 / x2 / x4 / x8）
    - 在任意lane上接收到的不是EIOS或者SKP OS
- 如果从first 8 bytes中infer到一个EIEOS，随着EIEOS的接收，Data Stream结束，并且在LTSSM允许的时候，**link必须进入recovery状态**。
- 如果从first 8 bytes中infer到一个EIOS，link准备进入已经协商好的低功耗状态，即L0p或者L1或者L2
- 如果下列条件符合，则发生**降lane**，且data stream继续传输（其实就是以一个更窄的link继续工作）：
  - 在一些active lanes上infer到了EIOS（从最初的8 bytes中）
  - 剩下的x1 / x2 / x4 / x8 lane上infer到了SKP OS
  - link启用了L0p
  - 在L0p下详细的升/降lane流程见后续的文档

在64.0 GT/s下，Data Stream中的Ordered Set处理流程如图所示

![image-20240427170055139](./assets/image-20240427170055139.png)

## Data Stream in Flit Mode
在Flit Mode下，flit size为256 Bytes。分配如下：
- 236 Bytes TLPs，即Byte 0～235
- 6 Bytes Data Link Layer Payload，即Byte 236～241的DLP，注意不是DLLP，结构不一样的
- 8 Bytes CRC，即Byte 242～249
- 6 Bytes ECC，即Byte 250～255。其中有3组ECC。

其中，Flit仍然**以Byte为单位**分配到各个lane上。
- 在64.0 GT/s及以上速率中，每个Byte即代表一个Symbol
- 在8.0 / 16.0 / 32.0 GT/s中，每个Byte对应128b/130b编码的data block中的一个symbol
- 在2.5 / 5.0 GT/s中，每个byte使用8b/10b编码转为10 bit symbol

采用8 Bytes的CRC来保护TLPs和DLLPs。采用6 Bytes的ECC来保护包括CRC在内的整个Flit。尽管只有在64.0 GT/s即以上速率，使用PAM4编码时，由于高FBER，整个flit需要ECC的保护，但是为了保证连续性，在**更低的速率下**仍然启用了ECC保护。

在link layer retry机制中，仅仅重发TLP，DLP不重发。因此，每一个flit（DLP）中的link layer payload根本不进入Tx侧的link layer retry buffer中。**重发的Flit中的DLP field仍然反映了最新的DL层payload**。因此，DLP可能会丢失，但是这个影响不大，因为DLP更新credits的方法是使用当前指针位置，而非指针变化量。

FEC是一种3路分离的ECC。每个ECC code能够修复单个Byte错误。因此，在16bit以内的burst error不会影响超过一个Byte

接收方和发送方的结构如图所示，注意使用不同编码方法时结构的不同：

![image-20240505162019313](./assets/image-20240505162019313.png)

![image-20240505162041367](./assets/image-20240505162041367.png)

## Bytes in Flit Layout
### TLP Bytes in Flit 
Flit中的TLP Bytes携带Transaction Layer的TLPs。因为在Flit Mode下，不支持STP Token，因此这些TLP纯由TL层输入。无论有没有TLP要发送，都要填上TLP。根据TLP的长度和分布位置，**一个TLP可能会被分到多个Flits中去**。下面是TLP的规则：
- 当TL层没有TLP发送的时候，它发送NOP TLP，其长度为**1 DW**，即4 Bytes。一旦NOP TLP被安排发送，则其必须使用**NOP TLP填充flit到4 DW对齐边界或者Flit结束（填充到的是Flit中的4DW边界，与实际的link宽度无关）**
- NOP TLPs不消耗任何credits
- Flit Mode下的TLP，有预先确定好的区域，来提示TLP的长度和是否有TLP Prefix存在，其实就是TLP包，第二章讲的那个
- 下列rules适用于Flit Byte 0～127或者Flit Byte 128～235中存在的non-NOP TLPs：
  - 不允许存在超过8个TLP，包含partial TLPs（即**每一半不允许包含超过8个TLPs**）
  - 可选检查，如果checked，在记录为Data Link Protocol Error
- 如果Flit结尾处的一个TLP通过**Flit_Status**设置为**poisoned**或者**nullified**，而且该TLP延续至后续的Flits，则这些Flits都必须通过**Flit_Status**设置为poisoned或者nullified，以便**TLP结尾所在的Flit**通过**Flit_Status**设置为poisoned或者nullified。因此，receiver**允许仅仅检查TLP结尾所在的那个Flit的poison bit**
- 如果Flit是valid，则通过Flit_Status来设置为nullified的TLP会被receiver丢弃，但是credits必须被释放（因为发送出来时，Tx侧的credits指针已经更新了）
- 使用EDB结尾，或者被通过Flit_Status来设置为nullified的TLP，**它后面只能接NOP TLP**，直到Flit结束。
### DLP Bytes in Flit
在每个Flit中，都有6 bytes被专门分配，用来传输在Non Flit Mode下由DLLP包含的信息。Flit中的DLP bytes大部分沿用了DLLP机制和格式，同时做了一些优化，来更有效的传输Ack/Nak flit，optimized credit relase，以及传统的DLLP。DLP中加入了一个encoding，来为PRH，PRD，NPRH传输Update_FC credits。DLP field如图所示：

![image-20240505162633434](./assets/image-20240505162633434.png)

在下表中，定义了**3种Flit Type**，并且指出了这三种Flit中的TLP field和DLP field的内容：

![image-20240505162707456](./assets/image-20240505162707456.png)

这三种Flit Type都有同样的CRC和FEC机制。区别在于它们携带的TLP和DLP Bytes类型不同。

- IDLE Flit，在DLP field中，使用sequence number 0。且replay command 00b。
  - 仅仅在128b/130b或者1b/1b编码中，在SDS OS + Control SKP OS之后，能够发送IDLE Flit 
  - 在8b/10b编码中，IDLE Flit作为TS2 OS的总结被发送。或者在最后一个TS2 OS后的SKP OS之后发送。在8b/10b编码中，通过SKP OS后有没有COM，来判断到底是OS还是Flit
  - 对于在所有FC/VC中都advertise infinite credits的device，如果没有DLLP需要发送，则允许使用NOP DLLP来填充DLP 2～5
- Valid Flit的定义：
  - 在performing FEC correction之后，通过了CRC check
  - none of ECC groups in the FEC report an uncorrectable error
  - Flit Usage和Flit Status fields没有使用reserved encoding
- FEC-Correctable Flit是指一个valid flit，它在performing FEC correction之后，通过了CRC check
- FEC Uncorrectable error是指：
  - 在FEC correction之后，仍然被CRC探测到的错误
  - FEC中的ECC Group汇报的uncorrectable error
  - 这些错误将导致NAK和replay
- Valid non-IDLE Flit指的是一个NOP Flit或者一个Payload Flit。
对于DLLP Payload，采用Optimized_Update_FC来发送performance critical NPRH / PRH / PRD credits。

有关Retry机制的内容，暂时跳过
### CRC Bytes in Flit
### ECC Bytes in Flit
### Ordered Set insertion in Data Stream in Flit Mode
本段描述SKP OS插入的频率问题。对于2.5 GT/s和5.0 GT/s，SKP OS插入的频率不变（1180～1538 symbols），但是**仅仅在Flit边界处插入**。其他的OS仅仅在L0p状态下的降lane和升lane出现。

对于8.0 GT/s及以上是速率，适用于以下规则：

- 在SDS OS之后，必须立即接上一个Control SKP OS，来表示Data stream的开始，而不管上一个Control SKP OS是什么时候发送出去的
- 当Data Stream开始之后，OS必须以固定的间隔被发送
  - 如果Data stream继续，SKP OS必须在所有active lanes上发送
    - 在L0p状态下的升lane：对于即将激活的lane，必须在所有**将要激活的lane上**发送一个SDS OS，而后在所有启用的lane上发送SKP OS。而后Flit将在更宽的lane上传输。
  - 如果link将要进入low-power state，则必须在所有active lanes上发送一个EIOSQ
  - 如果link需要进入Recovery，在link的所有configured lanes上，必须发送一个EIEOS
  - 如果link进入Recovery，在第一个EIEOS传输之后，必须发送合适的SKP OS

SKP OS的插入间隔如下：

![image-20240505164631760](./assets/image-20240505164631760.png)

Flit Mode下的consecutive SKP OS：无论采用128b/130b编码还是1b/1b编码，对于给定的Link Width和时钟模式，在Flit Mode下，Data Stream中的两个连续的SKP OS总是等距的。例如，对于SRIS Mode下的x1 link，采用128b/130b编码，SKP OS总是在每两个Flit后发送。占据32个data blocks。但是，对于采用8b/10b编码的传输而言，并不能保证这种等距。因为在8b/10b编码下，SKP的发送间隔为1180～1583 symbols（non-SRIS） / 小于154 symbols（SRS）。因此，有时它在每一个Flit后发送一个SKP OS，有时它在一个Flit后发送两个背靠背的SKP OS