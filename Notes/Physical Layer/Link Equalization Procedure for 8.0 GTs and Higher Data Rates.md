# Link Equalization Procedure for 8.0 GT/s and Higher Data Rates  

本文描述8.0 GT/s以及更高的速率中使用的链路均衡技术及其协商过程。

链路均衡这一步骤，允许components来调整每条lane的transmitter和receiver的setup，以**提升信号质量**，并且满足8.0 GT/s以上信号传输的电气需求。所有与LTSSM有关的lane（**当前正在工作的lane / 在未来由于升lane而有可能工作的lane**）都必须参与链路均衡的步骤。当速率第一次切换到任意8.0 GT/s以上的速率时，必须执行链路均衡步骤（其实是刚刚切换到目标速率，就进Recovery.Equalization进行链路均衡），**除非link两端所有components都advertise不需要链路均衡（这是通过link两端的一种协商机制来协商该link不需要进行均衡）**。链路均衡必须实现合适的transmitter setup，以满足transmitter在本次连接中（LinkUp=1b）未来所能碰到的**所有操作情形和速率**。在任何速率下，components都不允许**以reliable operation的理由**要求重复链路均衡的过程，尽管重均衡有可能会因为其他原因而出现。Components必须存储在链路均衡过程中协商好的transmitter setups，并且在未来的8.0 GT/s及以上速率的操作中使用这些设置（其实就是Tx的FIR的那几个参数，和Rx的几个参数）。即使在链路均衡过程结束之后，Components仍然允许**微调receiver的setup**，只要这些微调不会使得link变得不可靠（满足不了chapter 8的电气要求）或者进入recovery状态。

如果link两端的components在它们的TS1 / TS2 OS或者Modified TS1 / TS2 OS中advertise不需要链路均衡，则链路均衡的步骤能够被跳过。例如，如果一个component支持32.0 GT/s及以上的速率，并且能够**使用上一次链路均衡的结果来稳定运行**，或者**不需要链路均衡就能稳定运行**，那么它可能会选择advertise它对任何5.0 GT/s以上的速率都不需要链路均衡。

链路均衡的步骤可以自动发起，也可以由软件发起。强烈建议components对于5.0 GT/s以上的，自己想要工作的速率，使用自动链路均衡机制。但是，**在某些速率下**不采用自动链路均衡机制的components必须确保在工作于这些速率之前，**基于软件的机制**能在这些速率下进行链路均衡操作。

在通常情况下，在5.0 GT/s以上的速率中，只有低速率下已经完成了链路均衡，才允许在更高速率下执行链路均衡（这保证了在高速率下工作不稳定时，可以退回更低速率工作）。但是，spec提供了一种可选的机制，当link两端的components都支持32.0 GT/s或者以上速率时，直接从最高的NRZ速率（即32.0 GT/s）开始进行链路均衡（link两端的components要**使用TS1 / TS2 OS或者Modified TS1 / TS2 OS来advertise自己支持这一机制**）。当启用了该机制，且link两端的components都成功协商了该机制时，对8.0 GT/s和16.0 GT/s不执行链路均衡，而是对32.0 GT/s及以上速率都执行链路均衡（即从32.0 GT/s开始进行链路均衡）。对于5.0 GT/s以上的，没有执行链路均衡的速率（即8.0 GT/s和16.0 GT/s），link将**不会**工作于这些速率上，而且也**不会将它们advertise**为自己支持的最高运行速率。例如，一个link首先在2.5 GT/s下训练到L0，而后进入Recovery，在32.0 GT/s下进行链路均衡，而后再进入Recovery，在64.0 GT/s下进行链路均衡，从而跳过了8.0 GT/s和16.0 GT/s下的链路均衡。在这种情况下，link期待的工作速率仅仅为**2.5 / 5.0 / 32.0 / 64.0** GT/s。如果在32.0 GT/s或者64.0 GT/s下链路均衡失败（已经进过再均衡的尝试），**link需要在更低的速率下进行链路均衡**，则Downstream Port需要**停止advertise Equalization Bypass to Highest NRZ Rate Support**，并且确保**link返回2.5 GT/s或者5.0 GT/s的工作状态**。而后执行**常规的链路均衡流程（就像skip over equalization to the highest common data rates从未支持过一样）**。

**下面是Downstream发起的切换到更低速率并且进行重新均衡的流程。**如果在32.0 GT/s下的链路均衡是成功的，但是link在该速率下不能稳定运行，Downstream Port允许**发起到任意更低的速率的切换流程**，而后在该速率下进行链路均衡（如果需要的话），注意执行链路均衡必须符合对应流程要求。当在8.0 GT/s下执行链路均衡的步骤时，Upstream Port必须**disable** **Equalization Bypass to Highest NRZ Rate Support**。如果在更低速率下的均衡流程是软件驱动的（注意，并非所有的均衡流程都是自动发起的，有的是软件驱动的），则必须进行以下操作：

1. set the **Equalization Bypass to Highest NRZ Rate Disable** and **No Equalization Needed Disable** register bits to 1b，即disable bypass equalization
2. set the **target Link speed** such that the Link will be operational at 2.5 GT/s or 5.0 GT/s，即切速度
3. set the **target Link speed** to equalize at the lower rates starting with 8.0 GT/s onwards，即从8.0 GT/s开始进行链路均衡

注意，我们采用寄存器来控制bypass equalization操作。**Equalization Bypass to Highest NRZ Rate Disable bit**为1时，禁止在TS1 / TS2 OS中advertise Equalization Bypass to Highest NRZ Rate Support。

另外一种可选的机制是**跳过整个链路均衡的过程**，并且直接跳向最高的支持速率，即**No Equalization Needed Mechanism**。当link两端的components都支持32.0 GT/s或者更高速率，且**在TS1 / TS2或者Modified TS1 / TS2中advertise支持这一机制时**，则执行该机制。该步骤适合以下情况：
- 一个component能够从之前的链路均衡结果中恢复出**所有速率所需要的**均衡设置和其他电路设置
- 该component对5.0 GT/s以上的所有速率都不需要链路均衡过程

注意，当寄存器中的**Equalization Bypass to Highest NRZ Rate Disable bit**设置为1时，**不允许**advertise该机制（即只能在启用**Equalization Bypass to Highest NRZ Rate**的情况下，才能启用**bypass all equalization**）。强烈建议64.0 GT/s及以上速率的components都支持这一mechanism，尤其是在**上电之后，已经执行过一次链路均衡**的情况下。该机制有助于component从**deep power saving state**中更快的恢复，因为这样恢复时只要做Bit Lock和Flit Align即可，不用重新Equalizaition了。

如果link一端的component 使用TS1 / TS2 OS来advertise **No Equalization Needed**，而另一端的component使用TS1 / TS2 OS来advertise **Equalization Bypass to Highest NRZ Rate Support**。则整个link将工作于**Equalization Bypass to Highest NRZ Rate Support**。在Modified TS1 / TS2 OS中，设置**No Equalization Needed bit**为1b的情况下，必须同时设置**Equalization Bypass to Highest NRZ Rate Support**为1b。如果link一端的component 使用Modified TS1 / TS2 OS来advertise **No Equalization Needed**，而另一端的component使用Modified TS1 / TS2 OS来advertise **Equalization Bypass to Highest NRZ Rate Support**，则整个link将工作于**Equalization Bypass to Highest NRZ Rate Support**。如果一个component advertise的Modified TS1 / TS2中，设置**No Equalization Needed bit**为1b，但**Equalization Bypass to Highest NRZ Rate Support**为0b，则行为未定义。如果link两端成功协商No Equalization Needed，则components允许从任意速率直接切换到64.0 GT/s，无需经过32.0 GT/s，因为不需要执行均衡过程。

在以下条件满足时，执行自动均衡：
- 在最初的Link协商中（当LinkUp被设置为1b时），link两端的components通过TS1 / TS2 OS来advertise，它们至少支持8.0 GT/s的速率
- Downstream Port对5.0 GT/s以上的某个速率，选择执行链路均衡步骤。

虽然**不推荐**，但是Downstream Port可以选择仅仅在5.0 GT/s以上的**某些速率**执行自动链路均衡。在这种情况下，必须使用**软件驱动的链路均衡流程**来覆盖没有启用自动均衡的那些速率。

在自动均衡机制中，在进入L0状态后，无论当前的link速率是多少，如果需要进行链路均衡，则在链路均衡完成前，**任何component都不允许发送除了NOP2 DLLP之外的DLLP**。在所有需要使用自动均衡机制来进行链路均衡的速率下，所有lane的receiver和transmitter的setup都已经调整完成，链路均衡才视为完成。Downstream Port负责启动速率切换，切到需要执行链路均衡的速率。在任何链路均衡过程中（无论是自动的还是软件驱动的），Downstream Port都**不允许advertise比需要被执行链路均衡速率更高的速率的支持**（因为这样就切到更高的速率了）。举例如下：

一个Link，需要在8.0 GT/s和16.0 GT/s下执行链路均衡。Downstream Port通过**不advertise任何高于8.0 GT/s的速率**，从而进入Recovery，来对8.0 GT/s执行链路均衡。当**Equalization 8.0 GT/s Phase 3 Successful bit** and **Equalization 8.0 GT/s Complete bit** of the **Link Status 2 register** 都设置为1b时，则8.0 GT/s的链路均衡成功。在成功切换到8.0 GT/s并由Recovery进入L0之后，Downstream Port需要从L0再次切换到Recovery，并且advertise 16.0 GT/s data rate support（**即使实际上它支持32.0 GT/s**），将速率切换到16.0 GT/s并且执行16.0 GT/s下的链路均衡。因为如果一开始就advertise最高速率的话，则会直接切到最高速率了，其他速率下就不能进行链路均衡了。

如果Downstream Port探测到均衡问题，或者Upstream Port发出了重新均衡的请求（**将发送的TS2中的Request Equalization bit设置为1**，注意重新均衡的请求时Upstream Port发起的），downstream在工作于均衡失败的速率，或者在更高速率执行链路均衡之前，可能会**在当前速率**重新进行链路均衡。在同一速率下连续执行重新均衡的次数由具体实现决定，但必须是有限的。如果在最初的链路均衡，和后续的有限次重新均衡之后，link还是不能在指定速率下稳定运行，那么link必须降到**更低的速率**。

使用自动均衡的component，在均衡过程完成之前，**不允许启动任何autonomous降lane过程（注意，autonomous降lane，意思是因为节省功耗等原因，而非工作不稳定的原因，来执行降lane）**。对于一个Upstream Port，在它从Downstream Port接收到一个DLLP（NOP2 DLLP除外）之前，不允许发送任何DLLP出去（NOP2 DLLP除外）。如果Downstream Port进行重新均衡，则在均衡完成之前，它不允许发送任何DLLP出去（NOP2 DLLP除外）。一个Downstream Port可以根据它自己的需要，或者Upstream Port的request，来执行重新均衡。注意，重新均衡可能会影响到软件对link / device status的判断。

自动均衡过程中的DLLP Blocking：当针对8.0 GT/s及以上速率使用自动均衡机制时，Downstream Port需要阻止DLLP的传输，直到**使用自动均衡机制的所有速率都完成链路均衡**。而对于Upstream Port，它需要阻止DLLP的传输，直到它接收到一个Downstream Port传来的DLLP。如果link两端的compones都advertise它们支持16.0 GT/s或者以上的传输速率，但Downstream Port仅仅在8.0 GT/s上使用自动均衡机制，则它仅仅需要**在8.0 GT/s链路均衡完成之前**阻塞DLLP的发送。同样的，如果link两端的compones都advertise它们支持32.0 GT/s的传输速率，但Downstream Port仅仅在16.0 GT/s上使用自动均衡机制，则它仅仅需要**在16.0 GT/s链路均衡完成之前**阻塞DLLP的发送。如果Downstream Port从L0延迟进入Recovery，而此时由于所有速率下的链路均衡还未完成，对DLLP的阻塞会造成Downstream Port在L0下infer Electrical Idle time或者DLLP timeout，从而使得Downstream Port从L0切入Recovery进行retrain。这两个timeout都不应该被视作error，且导致的link retrain不会对正常的链路工作产生影响

本段中描述的DLLP limitations不适用于NOP2 DLLPs。当工作于Flit Mode时，进入recovery必须发生于Ordered Set边界处，当执行自动均衡时，在所有速率的链路均衡都完成之前，有可能会发送Flit出去。这些Flit中会使用NOP2 DLLPs。注意，NOP2 DLLPs的接收**不能被Upstream Port用于解锁任何non-NOP2 DLLPs的传输**。

当使用基于软件的均衡机制时，软件必须确保不会因为link在链路均衡的过程，而对任何正在传输的TLP产生影响。软件驱动均衡的步骤：
1. write 1b to the **Perform Equalization bit** in the **Link Control 3 Register**
2. write to the **Target Link Speed field** in the **Link Control 2 register** to **enable the Link to run at 8.0 GT/s or higher**，即设置目标工作速率
3. a write of 1b to the **Retrain Link bit** in the **Link Control register** of the **Downstream Port** to perform equalization，即手动开启retrain

在软件驱动的均衡过程中，针对8.0 GT/s及更高的速率下，在更低速率的均衡还未完成之前，且link不支持bypassing equalization to higher data rates的情况下，不允许运行于更高的速率。链路均衡成功的标志：
- 8.0 GT/s：**Equalization 8.0 GT/s Phase 3 Successful bit** and **Equalization 8.0 GT/s Complete bit** of the **Link Status 2 register** are both set to 1b
- 16.0 GT/s：**Equalization 16.0 GT/s Phase 3 Successful bit** and **Equalization 16.0 GT/s Complete bit** of the **16.0 GT/s Status Register** are both set to 1b
- 32.0 GT/s：**Equalization 32.0 GT/s Phase 3 Successful bit** and **Equalization 32.0 GT/s Complete bit** of the **32.0 GT/s Link Status Register** are both set to 1b
- 64.0 GT/s：**Equalization 64.0 GT/s Phase 3 Successful bit** and **Equalization 64.0 GT/s Complete bit** of the **64.0 GT/s Link Status Register** are both set to 1b

软件可以通过set link两端的**Hardware Autonomous Width Disable bit** of the **Link Control register**，或者采用一些其他的机制，来确保在set Downstream port的**Perform Equalization bit**之前，link工作于满宽状态（即在执行链路均衡之前link要处于满宽状态，其实就是暂时禁用Downstream Port的低功耗降lane宽的功能）。启用了autonomous width downsizing的component负责通过在将**Hardware Autonomous Width Disable bit**设置为1的**1ms之内**，将link导向Recovery和Configuration，将link升lane到满宽工作状态（其实就是不允许因为低功耗降lane宽了，则必须切到满宽）。如果一个Upstream Port，它在最初没有advertise 8.0 GT/s data rate，16.0 GT/s data rate，32.0 GT/s data rate，或者64.0 GT/s data rate，而且对于这些没有advertise的rate，不参与自动均衡流程，那么它相关的软件在指示Upstream Port进入Recovery，advertise先前没有advertise的data rate，并且发起速率切换之前，必须确保在均衡过程中，不会影响到正在发送的TLP。随后，Downstream Port会在最初的速率切换过程中，发起均衡流程，注意切换的目标速率就是Upstream Port转到Recovery时所advertise的速率

对于Upstream Port，它们需要在Recovery.RcvrLock state检查均衡问题。但是，对于Downstream Port和Upstream Port，它们都允许在任何时刻使用与具体实现有关的特殊机制来检查均衡问题。如果一个Port检测到了均衡问题，它将进行如下操作：
- 8.0 GT/s：**Link Equalization Request 8.0 GT/s bit** in the **Link Status 2 Register** is set to 1b.
- 16.0 GT/s：**Link Equalization Request 16.0 GT/s bit** in the **16.0 GT/s Status Register** is set to 1b.
- 32.0 GT/s：**Link Equalization Request 32.0 GT/s bit** in the **32.0 GT/s Status Register** is set to 1b.
- 64.0 GT/s：**Link Equalization Request 64.0 GT/s bit** in the **64.0 GT/s Status Register** is set to 1b.

除了set这些对应的register bit为1b之外，一个Upstream Port必须将LTSSM导向Recovery，而后进入Recovery.RcvrCfg。而且在Recovery.RcvrCfg状态下，请求合适速率下的均衡（通过将**TS2 OS**中**Request Equalization bit**设置为1b，且将**Equalization Request Data Rate bits**设置为出错的速率）。如果请求均衡，那么对于每个探测到的错误，仅仅请求一次。当请求均衡的时候Upstream允许**set the Quiesce Guarantee bit to 1b**，从而告知Downstream Port在1ms内开启的均衡进程不会对正常操作造成任何副作用。

当Downstream Port接收到Upstream Port发送而来的均衡请求时（处于Recovery.RcvrCfg state，并且接收到了8个连续的TS2 OS，其中Request Equalization bit为1b），他必须执行以下操作之一：

* 在完成从Recovery到L0的切换后的1 ms内，在requested data rate上，发起一次链路均衡。其中，requested data rate由接收到的Equalization Request Data Rate bit决定
* 根据速率拉起对应的寄存器bit，这是软件驱动的均衡流程
  * link status 2 register中的Link Equalization Request 8.0 GT/s
  * 16.0 GT/s Status Register中的Link Equalization Request 16.0 GT/s
  * 32.0 GT/s Status Register中的Link Equalization Request 32.0 GT/s
  * 64.0 GT/s Status Register中的Link Equalization Request 64.0 GT/s

仅仅当Downstream能保证发起链路均衡过程不会对自己的操作以及Upstream Port的操作产生影响时，才允许Downstream Port发起链路均衡。允许Downstream Port使用接收到的Quiesce Guarantee bit来决定执行链路均衡流程会不会对Upstream Port的操作产生影响

如果一个Downstream想要发起一次链路均衡，而且能保证链路均衡的流程对自己的操作不会产生副作用。但是它不能直接决定链路均衡的流程是否会对Upstream Port的操作产生副作用，则它允许请求让Upstream Port来发起一次链路均衡。对于Downstream Port，它先进入**Recovery**阶段，而后在**Recovery.RcvrCfg**中，将它传输的TS2 OS中的Request Equalization bit设置为1b，将Equalization Request Data Rate bit设置为想要的data rate，将Quiesce Guarantee bit设置为1b。当一个Upstream Port从Downstream Port收到一个这样的均衡请求之后（它处于Recovery.RcvrCfg state，并且接收到了8个连续的TS2 OS，其中Request Equalization和Quiesce Guarantee bit都为1b），允许静默自己的操作，并且准备在Downstream Port请求的速率下进行链路均衡。而后，Upstream Port在该速率下进行链路均衡，且将Quiesce Guarantee bit设置为1b。对于Upstream的相应时间没有要求，但是它应该尽可能快的响应。如果一个Downstream Port发出了一个上述的Request，而且从Upstream Port接收到了一个上述的response，则它执行以下操作之一：

* 如果此时，它仍然能够确保执行速率均衡不会对自己的操作有副作用，则它必须在在完成从Recovery到L0的切换后的1 ms内，在协商好的速率发起链路均衡
* 根据速率拉起对应的寄存器bit，这是软件驱动的均衡流程
  * link status 2 register中的Link Equalization Request 8.0 GT/s
  * 16.0 GT/s Status Register中的Link Equalization Request 16.0 GT/s
  * 32.0 GT/s Status Register中的Link Equalization Request 32.0 GT/s
  * 64.0 GT/s Status Register中的Link Equalization Request 64.0 GT/s

Using Quiesce Guarantee Mechanism：当Data Link Layer处于DL_Active之后，可能会因为执行链路均衡的流程而产生side effects。例如，链路均衡所花费的时间，可能会导致Completion Timeout error。因此使用Quiesce Guarantee information来指导Port，让它决定是否执行链路均衡的请求。

在汇报了一个均衡错误之后，component可能工作于更低的速率，这是通过**timeout后进入Recovery.Speed**或者**直接发起速率切换，到更低速率**来实现的。任何为了实现链路均衡而进行的速率切换都不受200ms的限制。有时，为了重新进行链路均衡，必须**将速率切换到一个中间速率**。例如，如果Downstream port想要在16.0 GT/s进行链路重均衡，而且**不支持bypass equalization**，且当前速率为2.5 GT/s或者5.0 GT/s，那他必须**先切速度到8.0 GT/s**（除非需要，否则无需在8.0 GT/s下进行链路均衡），从8.0 GT/s开始，它才能进行16.0 GT/s的链路重均衡。每次通过recovery state，链路均衡过程只能被执行一次。因为在切换到需要进行链路均衡的速度之前，链路均衡的步骤实际上是在低于目标速度的一个速度下执行的

不同情况下的链路均衡需求：
- From 2.5 GT/s or 5.0 GT/s to 8.0 GT/s Equalization
  - 本段中描述的链路均衡流程对首次均衡 / 链路再均衡，以及自动均衡 / 软件驱动均衡均适用
  - Downstream Port**使用8b/10b编码**，发送Upstream Port每个lane使用的Transmitter preset values / Receiver preset hints。当向更高速率的data change已经协商完成，且向更高速率的data change还没有完成之前（即在**Recovery.RcvrCfg**状态下），执行速率均衡，使用EQ TS2 OS来交流这些values（其实就是在Downstream Port的EQ Phase 0发送的）。EQ TS2 OS具体发送的values如下：
    - 对于8.0 GT/s下的均衡，发送的是**Upstream Port 8.0 GT/s Transmitter Preset** and **Upstream Port 8.0 GT/s Receiver Preset Hint fields** of **each Lane Equalization Control Register Entry.**
  - 在向更高速率（在该速率下需要链路均衡）的切换完成之后，link双方都由Recovery.RcvrLock进入Recovery.Equalization。Upstream Port使用它收到的preset values来发送TS1 OS。如果Upstream Port的Tx使用的时reduced swing，则Preset values必须符合电气要求
  - 在向更高速率（在该速率下需要链路均衡）的切换完成之后，Downstream Port必须使用以下的preset values来发送TS1 OS（假设该preset values符合电气要求），其实就是对于Downstream Port自己，它Tx Preset Value就是自己register中配的：
    - 对于8.0 GT/s下的均衡：**Downstream Port 8.0 GT/s Transmitter Preset** and **optionally Downstream Port 8.0 GT/s Receiver Preset Hint** fields of **each Lane Equalization Control Register Entry**.
  - 为了执行链路重均衡，Downstream Port必须在2.5 GT/s或者5.0 GT/s下的Recovery.RcvrLock states中，通过EQ TS1 OS来请求速率切换，从而告知Upstream Port它需要在更高速率下重新进行链路均衡。如果一个Upstream port想要在更高速率下工作，且收到了一个Speed change为1的EQ TS1 OS，则它必须在recovery阶段advertise更高的速率。
- From 2.5 GT/s or 5.0 GT/s to 32.0 GT/s Equalization，其实和到8.0 GT/s的均衡流程类似，只是发送的Transmitter Preset值，以及Receiver Preset Hints值不一样而已
  - 仅仅当link支持bypassing equalization to higher data rates时，允许使用这种均衡机制。例如，在32.0 GT/s Capabilities register中的Equalization Bypass to Highest NRZ Rate Supported bit为1b，且32.0 GT/s Control Register中的Equalization Bypass to Highest NRZ Rate Disable
  - 本段中描述的链路均衡流程对首次均衡 / 链路再均衡，以及自动均衡 / 软件驱动均衡均适用
  - Downstream Port**使用8b/10b编码**，发送Upstream Port每个lane上的Transmitter preset values / Receiver preset hints。当向更高速率的data change已经协商完成，且向更高速率的data change还没有完成之前（即在Recovery.RcvrCfg状态下），执行速率均衡，使用EQ TS2 OS来交流这些values。EQ TS2 OS具体发送的values如下：
    - 对于32.0 GT/s下的均衡，EQ TS2 OS中发送的是**Upstream Port 32.0 GT/s Transmitter Preset** of the **32.0 GT/s lane Equalization Control Register Entry** corresponding to the lane。而**Receiver Preset Hint field为000b**
  - 在向更高速率（在该速率下需要链路均衡）的切换完成之后，在Recovery.Equalizaiton中，Upstream Port使用它收到的preset values来发送TS1 OS。Preset values必须符合电气要求
  - 在向更高速率（在该速率下需要链路均衡）的切换完成之后，Downstream Port必须使用以下的preset values来发送TS1 OS（假设该preset values符合电气要求），其实就是Downstream Port的Tx Preset Value，如果有来自Upstream Port的配置，则优先使用配置值，没有的话则使用自己寄存器中配置的值：
    - 如果在最近的Recovery.RcvrCfg中，收到8个连续的EQ TS2，且其中的Transmitter Preset values是有意义的，则必须使用这些EQ TS2中提供的Transmitter Preset values。
    - 否则，必须使用**Downstream Port 32.0 GT/s Transmitter Preset field** of the **32.0 GT/s Lane Equalization Control Register Entry** corresponding to the Lane
  - 为了执行链路重均衡，Downstream Port必须在2.5 GT/s或者5.0 GT/s下的Recovery.RcvrLock states中，通过EQ TS1 OS来请求速率切换，从而告知Upstream Port它需要在更高速率下重新进行链路均衡。如果一个Upstream port想要在更高速率下工作，且收到了一个Speed change为1的EQ TS1 OS，则它必须在recovery阶段advertise更高的速率。
- From 2.5 GT/s or 5.0 GT/s to 64.0 GT/s Equalization
  - 不支持这种均衡模式。注意，所有针对64.0 GT/s下的均衡流程，包括重均衡，**都必须在32.0 GT/s下进行**。
- From 8.0 GT/s to 16.0 GT/s Equalization, From 16.0 GT/s to 32.0 GT/s Equalization, From32.0 GT/s to 64.0 GT/s Equalization。与2.5 GT/s和5.0 GT/s下发起的均衡流程相比，其实就是Downstream Port在EQ Phase 0，即Recovery.RcvrConfig state中发送的TS种类不同而已
  - 本段中描述的链路均衡流程对首次均衡 / 链路再均衡，以及自动均衡 / 软件驱动均衡均适用
  - Downstream Port**使用128b/130b编码**，向Upstream Port交流每个lane上的Transmitter preset values。当向更高速率的data change已经协商完成，且向更高速率的data change还没有完成之前（即在Recovery.RcvrCfg状态下），执行速率均衡，使用130b EQ TS2 OS来交流这些values。130b EQ TS2所包含的preset values具体如下：
    - 对于16.0 GT/s的均衡：**Upstream Port 16.0 GT/s Transmitter Preset field** of the **16.0 GT/s Lane Equalization Control Register** Entry corresponding to the Lane.
    - 对于32.0 GT/s的均衡：**Upstream Port 32.0 GT/s Transmitter Preset field** of the **32.0 GT/s Lane Equalization Control Register** Entry corresponding to the Lane.
    - 对于64.0 GT/s的均衡：**Upstream Port 64.0 GT/s Transmitter Preset field** of the **64.0 GT/s Lane Equalization Control Register Entry** corresponding to the Lane.
  - **可选的**，Upstream Port也可以**使用128b/130b编码**，向Downstream Port交流每个lane上的Transmitter preset values。当向更高速率的data change已经协商完成，且向更高速率的data change还没有完成之前（即在Recovery.RcvrCfg状态下），执行速率均衡，使用130b EQ TS2 OS来交流这些values。具体的value值由具体实现决定。在向更高速率（在该速率下需要链路均衡）的切换完成之后，Upstream使用它接收到的preset value传输TS 1 OS。如果Downstream port在Recovery.RcvfCfg阶段没有收到preset values，在速率切换到更高的速率之后，它使用如下的preset来传输TS1 OS：
    - 对于16.0 GT/s的均衡：**Downstream Port 16.0 GT/s Transmitter Preset field** of the **16.0 GT/s Lane Equalization Control Register** Entry corresponding to the Lane.
    - 对于32.0 GT/s的均衡：**Downstream Port 32.0 GT/s Transmitter Preset field** of the **32.0 GT/s Lane Equalization Control Register** Entry corresponding to the Lane.
    - 对于64.0 GT/s的均衡：**Downstream Port 64.0 GT/s Transmitter Preset field** of the **64.0 GT/s Lane Equalization Control Register Entry** corresponding to the Lane.
  - 注意，这些preset values必须符合第八章的电气要求
  - 为了进行重均衡，Downstream Port必须在Recovery.RcvrLock阶段发送TS1 OS，且Equalization Redo bit设置为1，从而告知Upstream Port它想要重新均衡。如果一个Upstream port接收到了TS1，且TS1的speed change bit为1，Equalization Redo bit也为1，且它想要在更高的速率下工作，则它必须在Recovery阶段advertise更高的速率。
- Equalization at a data rate from a data rate equal to the target equalization data rate
  - 即在当前速率下，就对当前速率进行链路均衡。该情形只适用于链路重均衡。而且只适用于从8.0 GT/s向8.0 GT/s的重均衡，从16.0 GT/s向16.0 GT/s的重均衡，从32.0 GT/s向32.0 GT/s的重均衡。在这种情况下，链路均衡所使用的的initial preset的值，就是上一次对该速率进行链路均衡时用的值。注意，**不支持从64.0 GT/s向64.0 GT/s的重均衡**。

链路均衡的过程分为4个Phase。当link工作于8.0 GT/s或者更高速率下时，Phase信息使用**TS0 / TS1**中的**Equalization Control（EC）field**来进行传输。
- Phase 0
  - 当link**正在协商**到需要进行链路均衡的速率（在正式进入该速率**之前**）时，执行该Phase。其中，链路均衡参数的默认值由上面描述的值决定，其实就是寄存器内的值。对于Downstream Port而言，是在进Recovery之后进Recovery.RcvrCfg，这就是Phase 0。
  - 而对于Upstream Port而言，Phase 0是在Recovery.Equalization，当它接受到来自Downstream Port的TS1 OS，且误码率小于10^-4^，则转而发送EC-01b的TS1 OS，进入Phase 1。也就是说其实Downstream Port的Tx侧的微调是在Downstream Port Phase 1而Upstream Port Phase 0完成的
- Phase 1
  - 协商完成，开始链路均衡。两边交换TS OS，来**建立一个可靠的link**
  - 首先，在当前速率下，Link两端的components**使得link协商到足够交换TS1 OS的程度**，从而能够完成剩下的phase中对Transmitter / Receiver pair的微调。其实就是至少要把两边接收数据的误码率降到10^-4^以下
  - 对于 8.0 / 16.0 / 32.0 GT/s，在该Phase交换TS1 OS。
  - 对于64.0 GT/s，在该Phase中交换TS0 OS，并且必须保证在离开该phase时，每个port或者pseudo-port都**能够在Phase1和Phase2中，在Upstream方向可靠的交换TS0 OS**。
  - 对于Link而言，在link两端的component准备好进入下一个Phase之前，link应该能够工作于BER低于10^-4^的状态。同样，所有Transmitter使用上表中的参数预设值。
  - Downstream通过发送EC=01b的TS1 OS（8.0 / 16.0 / 32.0 GT/s）或者EC=01b的TS0 OS（64.0 GT/s）来指示phase1。而Upstream port，在**调整了它自己的receiver之后（如果需要的话）**，为了确保它能继续链路均衡的过程，接收这些TS，并且切换到Phase1，同理在phase1中，Upsteam port发送EC=01b的TS0 / TS1。
  - Downstream接收到来自Upstream Port的TS0或者TS1 OS，且EC=01b，并且评估误码率小于10^-4^，则它进入EQ Phase 2，开始发送TS1 OS，EC=10b。而Upstream Port收到TS1 OS，EC=10b之后，也进入EQ Phase 2，开始发送TS1 OS，EC=10b
- Phase 2
  - 在该phase中，Upstream port在每条lane上独立**调整Downstream port的transmitter**，同时**调整自己的receiver**，来确保自己接收到的bit流符合电气要求。
  - 如果link工作于8.0 / 16.0 / 32.0 GT/s，Downstream port和Upstream port都发送TS1 OS。
  - 如果link工作于64.0 GT/s，Downstream port发送TS0 OS，**直到它接收到一个TS0 OS**。而后，无论Upstream port是否发送TS0 OS，Downstream ports**转为发送TS1 OS**
  - Downstream port通过向Upstream发送EC=10b的TS0 / TS1（根据速率决定）来进入phase 2。
  - Downstream port advertise Transmitter coefficients和preset，在phase 1中，仅仅advertise preset，而在phase 2中，advertise preset and coefficients。接收到这些参数的Upstream Port可能会request不同的coefficient or preset values，并且会持续不断地评估每个组合，直到找到Downstream lanes的最佳组合
  - 当Upstream ports完成phase 2之后，在32.0 GT/s及以下速率中，它发送EC=11b的TS1 OS，在64.0 GT/s中，它发送EC=11b的TS0 OS，从而切换到phase 3，即Upstream Port首先进入Phase 3
- Phase 3
  - 在该phase中，Downstream port在每条lane上独立**调整Upstream port的transmitter**，同时**调整自己的receiver** settings，来确保自己接收到的bit流符合电气要求。**整个握手和评估过程与phase 2类似，除了EC=11b**。Downstream port通过发送EC=00b的TS1 OS来标志phase 3的结束（同时也是均衡过程的结束）

对于一个component而言，它使用的调整Link Partner的transmitter，以及使用自己的Receiver setup来评估Link partner的transmitter的setup的算法，由具体实现决定。一个component可能一次向任意数量的lanes request changes，并且对不同的lane，可能会request不同的settings。每一个requested setting都可能是一组preset或者一组coefficients，只要满足对应规则要求。每一个component都必须负责确保在fine-tune的结尾（对于Upstream Ports，为Phase2。对于Downsteam Ports，为Phase3），它的link partner都将transmitter settings设置为符合对应的电气特性的值，即能正常传输的值。

对于一个component，当它接收到来自link partner的request，要求调整自己这一侧的Transmitter参数，它必须评估该request。

1. request的是valid preset，且transmitter工作于Full swing mode，则request的内容必须反映在自己transmitter的setup上，并且反映在发送出去的TS1 OS的preset和coefficient field上（其实就是接收到对方发来的细调参数了，自己的Tx使用了这个参数，而后也在自己发送的TS1 / TS0 OS中echo back了这些参数）。
2. 如果request一个preset value，且transmitter工作于reduced swing mode，且requested preset是支持的，则request的内容必须反映在transmitter的setup上，并且反映在发送出去的TS1 OS的preset和coefficient field上（和full swing mode一致）。
3. 当工作于reduced swing mode，且如果不支持，则transmitter允许拒绝preset request。调整coefficient的request允许被接收或者被拒绝。如果接收了调整coefficient的request，则request的内容必须反映在transmitter的setup上，并且反映在发送出去的TS1 OS的coefficient field上。如果拒绝了调整coefficient的request，则自己的transmitter setup不会改变，但是发送出去的TS1 OS中必须反映出request的coefficient，并且将Reject Coefficient Values bit设置为1b（其实就是无论接不接受对方设置的参数，在自己发送的TS1 OS中都要把这些参数echo back）。
4. 对于调整coefficient的request，只有当他不符合约束时，才会被拒绝。

如果一个lane正处于Polling.Compliance / loopback或者Recovery.Equalizaiton的Phase0或Phase1，且接收到了一个transmitter preset value，其值为reserved或者unsupport value，则该lane允许使用任何支持的transmitter preset setting。在后续发送出去的compliance pattern或者OS中，需要在对应field中填充这些reserved或者unsupported value，虽然lane的Tx其实并没有使用这些参数。例如，如果一个Upstream的lane在Recovery.Rcvrcfg state接收到了EQ TS2 OS，且preset value 1111b（reserved），则它将自己的速率切换到8.0 GT/s之后，允许使用任意支持的transmitter preset value。但是，他必须在Recovery.Equalization中，在自己发送出去的TS1 OS中，将1111b作为transmitter preset value

在Loopback state中，由Loopback Lead负责通过在2.5 GT/s或者5.0 GT/s下发送的EQ TS1 OS中交流它想要Follower使用的Transmitter and Receiver settings，并且交流它想要测试的device在8.0 GT/s或者更高速率下发送的TS1 OS中的preset和coefficient settings。同理，如果通过TS1 OS进入8.0 GT/s或者更高速率下的Polling.Compliance，则正在进行测试的entity需要在EQ TS1 OS中发送合适的settings和presets给device under test。

下图展示了不同速率下的Equalization过程

![image-20240611223754472](./assets/image-20240611223754472.png)

![image-20240611223816201](./assets/image-20240611223816201.png)

![image-20240611223905777](./assets/image-20240611223905777.png)

下图展示了Equalization Bypass的过程

![image-20240611223618938](./assets/image-20240611223618938.png)

## Rules for Transmitter Coefficients
本节规定了Transmitter系数的一些规则，这些coefficients是用来调整FIR filter的。注意这些规则同时适用于advertised和requested coefficient settings

1. 有4个参数，即C~-2~ / C~-1~ / C~0~ / C~+1~ ，注意C~-2~仅仅适用于64.0 GT/s以及以上速率，其实就是当前发送的电压幅值，需要考虑当前值 / 下一个UI的值 / 前1个UI的值 / 前2个UI的值
2. 这4个参数的绝对值之和代表FS，即Full Swing，即总的电压幅值
   1. 对于Full Swing Mode，24 < FS < 63
   2. 对于Reduced Swing Mode， 12 < FS < 63
   3. 对于低于64.0 GT/s的速率，C~-2~设置为0
3. 在Phase 1中，一个transmitter advertise LF（Low Frequency）value
   1. 这一值定义了transmitter所能发送出去的最小的差分电压值。其大小为LF/FS * max differential voltage
4. 当发送针对link partner的transmitter的request时，必须满足以下要求
   1. |C~-1~| <= Floor(FS/4)
   2. |C~-2~| + |C~-1~| + |C~0~| + |C~+1~| = FS
   3. C~0~ - |C~-1~| + |C~-2~| - |C~+1~| >= LF
   4. |C~-2~| <= Floor(FS/8)

## Encoding of Presets
Transmitter和Receiver使用的Preset Hints的定义在第八章。下表展示了Transmitter和Receiver Preset Hint的编码，其实就是选取哪一组preset值的问题。有两种preset值，P0~P10用于8.0 GT/s，16.0 GT/s和32.0 GT/s，而Q0～Q10用于64.0 GT/s。尽管在这两组值中，有的是重叠的，但也存在这么一种情况，即同一个preset值在不同速度下对应不同的encoding。Receiver Preset Hints是可选的，而且仅仅为8.0 GT/s定义，因为在其他速率下的链路均衡中根本就没有Receiver Preset Hints。

![image-20240511141626081](./assets/image-20240511141626081.png)

![image-20240511141646503](./assets/image-20240511141646503.png)

64.0 GT/s下的量子化错误：

- 由于在64.0 GT/s下，拥有更紧的preset tolerance，某些FS值会导致比spec规定的更大的量子化错误。具体实现必须补偿量子化错误。