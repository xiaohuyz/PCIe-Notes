# Link Training and Status State Machine (LTSSM) Descriptions
本章简要描述LTSSM中的各个states，具体的states切换如图所示。注意，对于LTSSM而言，所有的timeout values的值都是-0 / +50%，每个IP的具体的值，需要在验证时在VIP侧进行设置以告知VIP。在Fundamental Reset之后，所有的timeout value都必须被设置为特定值，而所有的counter值也必须被设置为特定的值。
## Detect Overview
该state的目的是为了检测link对端是否有device存在。
## Polling Overview
即轮询。在该阶段，port发送training Ordered Sets，并且对接收到的training Ordered Sets作出响应。在该state中，建立起bit lock和Symbol lock（即Symbol align），同时配置Lane polarity。

在Polling state中，包含了Polling.Compliance state。这个state是给test equipment用的，这些test equipment用来检测device中的transmitter和interconnect是否符合spec中描述的电压和时序要求。

Polling.Compliance同时包含了一个简化的互操作性测试方案，这用于一系列test & measure equipments。Polling.Compliance的这一部分通过如下方法进入：
- 在进入**Polling.Active**时，至少有1个component assert **Compliance Receive bit (bit 4 in Symbol 5 of TS1)**，且没有assert **Loopback bit (bit 2 in Symbol 5 of TS1)**
- set Compliance Receive bit的能力由具体实现决定
- 同时，也包含了速率切换的功能，从而使得测试能够在不同速率下进行

Polling.Compliance用于compliance test environment，在常规操作中，不能被进入，而且该state也不能以任何理由被disable。基于物理系统环境（即探测到对端有device，但是对端不发送OS出来，暗示对端是一个被动的测试负载）或者configuration register access机制来进入Polling.Compliance。任何其他的让一个transmitter发送compliance pattern出来的机制，由具体实现决定。
## Configuration Overview
在Configuration阶段，Transmitter和Receiver都按照协商好的速率来发送和接收数据。Port中的lanes通过**width and Lane negotiation sequence**来被配置为一个link，其实就是配置link的宽度。同时，在该阶段，发生了**Lane-to-Lane de-skew**，如果允许的话，**scrambling可以被disabled**，同时设置了**N_FTS**。在该阶段，后面能进入Disabled / Loopback state.
## Recovery Overview
在recovery阶段，Transmitter和Receiver都按照配置好的Link和Lane number，以及先前协商后好的速率来发送和接收数据。Recovery有如下功能：
- 如果需要的话，允许一个configured link改变data rate
- 重新建立bit lock / Symbol lock / block alignment / lane-to-lane de-skew，这一般是用于link的错误恢复
- set a new N_FTS value
- 进入Loopback / Disabled / Hot Reset / Configuration states
## L0 Overview
L0为常规的操作state。在该states下，data和control packets被发送和接收。注意，**所有的power management states都是从L0进入的**，其实就是从L0进入各种低功耗状态。
## L0s Overview
L0s是一个power savings state。
- L0s state在Flit Mode下不受支持。当一个Link工作于Flit Mode时，硬件必须忽略L0s Enable bit的值
- 当一个link中包含retimers时，不支持L0s
- 当工作于单独的参考时钟下，且启用了单独的SSC时，不支持L0s

L0s允许一个link快速进入power conservation state和从power conservation state中恢复，而无需经过recovery阶段

在接收到一个EIOS之后，进入L0s

从L0s退出，进入L0的过程，必须重建bit lock / symbol lock / block alignment / Lane-to-Lane de-skew

对于一个link而言，transmitter lane pair和receiver lane pair无需同时处于L0s。例如这么一个EP，它大部分时间都在接收数据，那么它的receive lane可能处于L0而transmitter lane则处于L0s
## L0p Overview
本节描述L0p，注意这是PCIe Gen 6新增的部分，它是L0 state的一个子状态，涉及低功耗和升降lane

L0p是L0的一部分，目的是作为一个power saving state。在**所有速率**下，L0p仅仅适用于**Flit Mode**。对于所有Ports，L0p support是可选的，但是强烈建议port支持L0p。对于支持Flit Mode的Pseudo-Ports（retimers），必须支持L0p。L0p状态允许一些lane处于active状态，而其他的lane处于electrical idle状态（这在以前的PCIe中是不被允许的）。在Flit Mode下，Link Upconfigure，即L0->Recovery->Configuration->L0，不再受支持。

L0p在宽度上是对称的。当一个port支持L0p时，它必须支持配置的最大link width内的所有合法的link width值， 例如x1，x2，x4，x8，x16。当Link处于L0p状态时，在每个方向上，至少要有一个lane是active的。L0p的协商发生于LinkUp=0b时的Configuration.Complete state，其实就是第一次训练到Gen1， flit mode时就协商好L0p support了。

Device Port Functions通过设置device capability 3 register中的L0p supported bit和Port L0p Exit Latency value，来指示L0p的支持

下面分别讲述硬件驱动和软件驱动的L0p request

对于一个port，如果L0p enable为1b，而Hardware Autonomous Width Disable为0b（即允许因为功耗原因而自动降lane），Target Link Width设置为111b，则该port允许在没有任何软件介入的情况下，通过发送Link Manangement DLLP来发起L0p requests。**注意上述条件**

而对于system software而言，它能够指导一个Port function来发起一个link width change，方法是发送一个CfgWr TLP给指定的function，将L0p Enable设置为1b，而设置对应的Target Link Width，注意target link width范围时000b - 111b，不允许111b，因为这个代码是留给硬件用的，即没有软件介入的情况下的动态lane宽度切换。强烈建议除了test / debug情形，system不改变Target Link Width。

如果一个link支持L0p，且它的link宽度已经通过进入Configuration state改变过了，则使用以下规则：

1. 在进入recovery state时，link将会回到最后一次进入L0时所配置的link宽度

2. L0p不能打开Configuration阶段所关闭的lanes（在configuration阶段关闭的lanes可能是由于工作可靠性的原因，而且在LinkUp=1b时，这些lanes与LTSSM无关了，因此在不重新configuration的情况下是不可能重新enable这些lanes的）

在支持L0p的情况下，任何进入electrical idle的lane，都必须停留在electrical idle状态至少T~TX-IDLE-MIN~。在electrical idle状态下，Tx侧的DC common mode voltage必须符合spec规定。而对于receiver而言，它必须至少等待T~TX-IDLE-MIN~，而后监测Electrical Idle Exit

当L0p被启用时，一个port可以通过在5个连续的Flits中，发送至多3个相同的Link Management DLLPs，来请求link width的切换。注意，即使被ordered sets隔开，Flit也被视为连续的。这种请求可以是up-size，即增加active lanes的数目，或者是down-size，即减少active lanes的数目，注意这种升/降lane和通过configuration state的那种不一样，通过configuration state的升/降lane是把lane加入/退出ltssm，这就意味着在传统的升/降lane流程中，降lane后要想重新升lane，则需要重新从L0进入Configuration，从而将新的lane与ltssm相关联。Link partner必须Ack或者Nak每个Valid Flit中的L0p request，除非它被要求ignore这些request。当接收到一个带着L0p request的valid Flit时，port必须在以下时间内回复L0p request：如果使用8b/10b编码，则为4us，其他情况下为1us。时间的计算从前。如果一个port请求了link width change，但是在以下时间内没有得到回复，则它可以选择重新发送这个请求，或者直接放弃：对于8b/10b编码，时间为8us，其他编码时间为4us。如果在5个连续的Flits中接收到了相同的L0p request，则应该返回相同的response。在responding一个request之后，如果以下条件有一个满足，则link partner必须认为request被abandoned，其实就是requseter确实是发起了升lane/降lane的L0p request，但是在request之后没有实际的动作：

* 对于一个up-size request：
  - 在response with Ack被发回去之后，requester没有在16us内发起升lane，并且没有在8us之内重发up-size request

- 对于一个down-size request with Ack response：
  - 在response with Ack被发回去之后，requester没有在4us内重发request，并且
  - 在response with Ack被发回去之后，requester没有在16us内发起降lane，且link partner自己没有通过发送EIOSQ来降lane（这种情况对应于Ack DLLP在传输过程中丢失或者损坏）
- 对于down-size request with Nak response：
  - 在response with Nak被发回去之后，requester没有在4us内重发request，其实就是request被Nak之后，requester就不再发送request了

对于L0p，必须遵循以下规则：
- 如果Link Control Register中的Hardware Autonomous Width Disable bit为1（即不允许link因为低功耗的原因而降lane），或者L0p enable bit为0，则port必须：
  - 不能请求down-size
  - 如果该link没有工作于fully configured width，则必须发起up-size request，来请求full configured link width（其实就是L0p下的那一套降lane流程被禁用，不再允许同一个ltssm内，一部分lane处于工作状态，而另一部分lane处于electrical idle）
  
- L0p的Ack / Nak规则，其实主要涉及L0p request的接收和分析问题，需要结合Link Management DLLP的具体field来分析：
  - 如果L0p.Priority为0b（意味着这是一个标准优先级的L0p request），则允许一个port Nak一个降lane的L0p request
  
  - 如果一个port因为热保护或者可靠性原因需要降lane，则它必须在L0p request中将L0p.Priority设置为1（来告知link partner这是一个高优先级的L0p request，这次的降lane不是普通优先级的因为功耗降lane）。对于Link partner，如果它没有在requesting an L0p width change，而且也没有想在下一个Flit中request an L0p width change，则它必须Ack priority L0p requests。如果它确实想要在下一个Flit中request an L0p width change，则请求必须在100ns内出现在transmitter pin上，且L0p.Priority为1。在up-size request中set L0p.Priority，则结果未定义。
  
    > 总结就是L0p.Priority只用于降lane的L0p request，用来告诉link partner这次降lane是因为热保护或者工作可靠性原因，因此这是一次高优先级的降lane，如果link partner此时没有恰好也在请求宽度切换，则link partner必须Ack这次降lane request
  
  - 当阻止link工作于full width的因素消失后，当初负责发起降lane的port，负责发起升lane
  
  - 如果link两端同时发起link width change请求，则根据以下规则来判断哪个请求获胜。注意，如果一个port接收到了link width change request，并且它自己将要在100ns内把自己这边的请求放在transmitter pin上，则认为这是同时的，也就是说这个同时的定义，由100ns的时间窗口。竞争失败的port，必须使用response with Ack来响应另一个port的link width change request：
    - 如果两个port都set了L0p.priority：
      - 如果两个port都请求了同一个width，则Downstream port wins
      - 否则，请求lower width的port wins
    - 如果仅仅有一个port set了L0p.priority，则它wins
    - 如果两个port都没有set L0p.priority
      - 如果两个port都请求了同一个width，则downstream port wins
      - 否则，请求lower width的port wins
  - 允许一个port Nak一个降宽的request，如果该request没有set L0p.priority。其实就是允许link partner Nak因为低功耗而发出的L0p downsize request
  - 如果一个Port已将ack一个L0p request，并且它的transmitter lanes已经处于，或者正在切换到那个宽度，此时收到另一个L0p request，且请求的宽度与上一个ack的L0p request一致，则该Port必须ignore这个request
  - 如果LTSSM切换到Recovery，则link切换到configured link width，该width是最近一次从Configuration切换到L0时的width（其实就是没有L0p介入下的正常工作速率）。并且，该Link上所有有关的L0p transition states（例如width change request / response，以及对于这些行为的tracking例如从上次Ack Nak经过的时间）都会被重置，就像没有link width request发生过一样。

一旦一个请求link width change的L0p request被发起，它要么被complete，要么被anandoned

除非以下条件都满足，否则不允许发起一个针对新的Link Width的L0p request。其实就是在没有L0p link width resizing正在进行，且没有L0p request -> Ack/Nak握手正在进行时，才允许发起新的L0p request
1. 对于1b/1b或者128b/130b编码，在1us内，对于8b/10b编码，在4us内，没有link width resizing正在进行
2. 如果port已经abandoned一个L0p request，则至少要等待16 us
3. 上一个L0p request，在port ack之后， 被abandoned了

下面讲述L0p下的link resizing，包括down-size和up-size，这也是唯一的允许不同lane发送/接收不同种类的Ordered Sets的state

在link width down-size过程中，在Link width down-configure request被ack之后，每个port都必须独立进行以下steps：

- 在下一个安排好的SKP OS interval处，将要被turned off的lanes，将会发送能够EIOSQ来代替SKP OS，注意在此时，同一个Link上不同lane就开始发送不同种类的Ordered Sets了
- 在发送EIOSQ之后，发送EIOSQ的lanes进入Electrical Idle。而发送SKP OS的lanes继续发送Flits，而且使用的SKP OS插入周期调整为与新的link width相适应
- 如果requesting port发出了request，但是没有收到Ack，但是看到link partner发送EIOSQ OS了，则视为收到Ack了。这是为了防止link partner返回的含有Ack DLLP的Flit corrupted，在这种情况下link partner确实是发出去Ack了，因此它自己就开始downsize了，但实际上requester没收到Ack。
- 在down-size negotiation过程中，在调整lane to lane skew之后，所有将要被deactivated的lanes上必须同时收到一个EIOS，否则LTSSM将要进入Recovery（其实就是要进入Electrical Idle的所有lane中，lane与lane之间不对齐）
  - 对于一个Port，如果link没有在两个方向都没有切换想要的link width，则在发送EIOSQ的24ms之后，允许进入recovery

在Link width up-size过程中，必须执行以下steps：
- up-sizing action必须被requesting port发起。non-requesting port等待需要被activate的lanes上检测到Electrical Idle exit，而后才开始up-sizing actions
- 在本来就active的lanes上，data stream继续传输
- 对于那些将要被activate的lanes，必须遵循以下sequence：
  - 对于Receiver，SKP OS必须在和lane0上一样的时间进行安排，并且像lane0一样添加和删除同样数量的SKP OS。在8b/10b编码下，允许port截断TS1/TS2 OS，从而和lane0在相同的时间传输SKP OS
  - 首先，发送TS1 OS，必须满足spec对TS1 OS的要求。transmitter必须保证follow EIEOSQ的rule，且不会被SKP OS打断，即使它需要和active lanes一样发送SKP OS时（其实就是EIEOSQ不允许被SKP OS打断）。这可以通过恰当安排EIEOSQ的开始时间来实现，使得它不会出现在应该发送SKP OS的窗口内。但是如果一个SKP OS打断了一个EIEOSQ的进程，则transmitter允许重新启动EIEOSQ sequence。如果将一个EIEOSQ安排在SDS OS之前的256UI，则transmitter允许不发送那个EIEOSQ，或者延迟SDS，直到下一个SKP interval
  - 如果在所有需要被激活的lanes上接收到了8个连续的TS1或者TS2 OS，则transmitter需要按照SPEC的规则发送TS2 OS
  - 在所有需要被激活的lanes上，接收到了8个连续的TS2 OS，并且在接受到第一个TS2 OS之后至少发送出去16个TS2 OS之后，如果port速度为8.0 GT/s或者更高，则port发送一个SDS OS出去，而后发送下一个安排好的SKP OS（其实就是发送TS OS之后，等待下一个SKP OS需要发送的时候，先发SDS OS出去，而后发SKP OS，这样就开启数据传输了）
  - 如果速率为8.0 GT/s或者更高，则receiver必须在所有现在active的lanes，以及那些将要被active的lanes上（已经接收到8个连续的TS2 OS和SDS OS）使用SKP OS，来完成lane to lane deskew。从SDS之后的SKP OS之后，link就工作于新的宽度了，SKP OS的插入周期根据新的link宽度而定。
  - 如果从activation开始之后，已经经过24ms了，而lane仍然不是active lanes的一部分，则LTSSM必须进入recovery。其实就意味着升lane失败了

L0p是L0的一个reduced power sub-state。它与L1和L2是互斥的。对于一个port，如果正处于发起或者接受L1 / L2 request的过程中，则建议不要发起 / 更新或者回复L0p request。然而，L0p和L1的协商，或者L0p和L2的协商实际上是允许同时进行的，并且最终进入一个协商好的powerstates。最后到底进入哪个power states，取决于sequence of negotiations和port期望的目的。
### Link Management DLLP

Link Management DLLP的具体field如图所示

![image-20240804182306551](./assets/image-20240804182306551.png)

![image-20240804182321890](./assets/image-20240804182321890.png)

在以下情况下，receivers必须ignore Link Management DLLP：

- link工作于Non Flit Mode
- link management type field包含一个reserved value
- link management type field为00h，且L0p.cmd包含一个reserved value
- link management type field为00h，且L0p.cmd为L0p request，但是L0p Link Width field包含一个reserved value
- link management type field为00h，且L0p.cmd为L0p request ack或者L0p request nak，并且response payload包含了一个reserved value

L0p entry / exit times：对于那些处于electrical idle的lanes，它们期望有和L1类似的power savings和entry / exit times。power saving越激进，则将这些idle lanes切换回active时的exit latency越大。例如，那些idle的lanes所独有的PLL可以被关闭，这会造成更好的能量节省，但是重新激活这些lanes的时间将会变长。

receivers硬件应该使用Data Link Feature DLLP中的L0p exit latency来决定何时request L0p。这些值同时也存在于Local L0p Exit Latency和Remote L0p Exit Latency fields，因此能被软件读取。系统软件应该确保所有的retimers的延迟已经包含在Retimer L0p Exit Latency中。总的来说，对于存在于add-in card中的retimers，其exit latency应该包含在add-in card的Upstream Port的Retimer L0p Exit Latency中。而那些存在于系统底座上的retimers应该被包含在Downstream Port的Retimer L0p Exit Latency中。

下图展示了在一个x16 link上的L0p流程：
1. 在Configuration.Complete state中，在Link上启用L0p（通过两边的TS OS协商）
2. Upstream Port要求link downsize to x8，而DSP Ack了这个request
3. 在下一个安排传输SKP OS的时候，Lane 0-7继续传输，而lane 8-15发送EIOSQ，并且进入electrical idle。因此link此时有8个active lanes，而另外8个lanes处于idle state
4. 后来，Downstream Port request upsize to x16，但是同时Upstream Port request further downsize to x4。而x16的request被Ack，x4的请求被Nak，因此发生升lane行为。link training发生在Lane8-15，由DSP发起，Downstream开始交换TS1/TS2 OS，且在合适的间隔处，插入EIEOS。当lane8-15 ready时，在下一个安排的SKP OS边界之前，立即发送SDS，而后开始正常的传输
5. 而后，在所有的16lanes上发送SKP OS之后，link工作于x16 mode。注意尽管L0p在link两端是对称的，但是link两端可能在某一小段时刻内工作于不同的宽度。这和进入L1或者L0时的情形类似。

具体步骤如图所示

![image-20240804183117612](./assets/image-20240804183117612.png)

注意，如果link工作于2.5 GT/s，则EIEOS和SDS将不存在，而如果link工作于5.0 GT/s，EIEOS将存在，而SDS不存在。
## L1 Overview
L1是一个power savings state。

与L0s相比，L1 state允许额外的power savings，代价是额外的resume latency 

在接收到一个EIOS之后，Data Link Layer将device切换到L1
## L2 Overview
在L2 state下，能够十分激进的节省能量。在L2下，大部分Transmitter和Receiver都被关闭。主power和clocks也不能保证工作，但是AUX power总是可用的，这是用来支持唤醒功能用的。

如果特定的PCIe器件形态（例如m2）或者spec需要Beacon support，则支持wakeup capability的Upstream必须支持发送Bacon wake up signal，而Downstream Port必须支持接收Bacon wake up signal

在被Data Link Layer导向L2，且接收到一个EIOS之后，device切换到L2
## Disabled Overview
Disabled state的目的是允许一个configured link被disable，直到在进入disabled之后，被导出disabled，或者退出electrical idle（这一般是因为热插拔）。
## Loopback Overview
Loopback用于测试和错误隔离。在spec中，仅仅规定了Loopback state的进入和退出，关于Loopback的其他细节，都由具体实现决定。Loopback可以运行于单独的lane上，也可以运行于一整个配置好的link上。

Loopback Lead是**发起loopback**的component

Loopback Follower是looping back回data的component

在8b/10b和128b/130b编码中 ，Loopback使用TS1 / TS2 OS的Training Control Field中的bit2 (Loopback) 。在1b/1b编码中，Loopback使用TS1 / TS2 OS的Training Control Field中的bit 3:0的编码。

对于一个Loopback Lead而言，Loopback的进入机制由具体device决定，例如软件驱动的。

对于一个Loopback Follower而言，当接收到两个连续的TS1 OS，且TS1 OS中的Loopback bit置1，则进入Loopback状态

一旦处于Loopback状态，Lead能够发送任意的symbols（只要符合编码规则）。当处于Loopback状态时，data scrambling的概念也不再重要，因为所有发送出去的数据将会被loopback回来。

对于Loopback Lead而言，Data Link Layer如何告知Physical Layer进入loopback由具体实现决定。

## Hot Reset Overview
Hot Reset的目的是允许一个configured link以及与其相关的downstream device能够被使用in-band signaling来reset，其实就是使用inband 的TS OS来reset。