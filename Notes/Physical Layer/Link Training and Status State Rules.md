# Link Training and Status State Rules
本章描述LTSSM的细节。其中，由软件来监测各种各样的Link status bits，但是LinkUp的状态是由Data Link Layer来监测的，其实就是LinkUp这个bit是DL层用来开启自己的DLCMSM训练用的，对于软件是不可见的。下表展示了在LTSSM训练过程中，Link status bits的状态。同时，当允许汇报receiver error时，当link工作于8b/10b encoding，允许一个receiver在Lane Error Status Register中报告8b/10b error。

对于在configuration state下的LinkUp bit，因为有几种进入configuration的方法，因此该bit有2种可能性

1. 从Detect - Polling - Configuration。在这种情况下，LinkUp一直为0
2. 从其他states进入configuration，在这些情况下，LinkUp一直为1
上表中描述了Link Width和Link Speed是在何时确定的，以及是否能够report Receiver Error，同时描述了几个在硬件内部使用的寄存器（标志位），如下：
1. LinkUp，即描述Link是否已经进行过一次成功的训练
2. Link Training，指示Link是否在协商宽度和速率，这个bit仅仅在Configuration和Recovery State置1
3. Receiver Error，仅仅在指定的state中能report receiver error
4. In-Band Presence，指示link对端是否有device存在，仅仅在Detect state为0，其余state均为1
Receiver errors during configuration and recovery states：当LTSSM处于Configuration或者Recovery state时，允许Receiver Errors被set，这是为了汇报在这些states中处理packets时发生的link errors。例如，当一个TLP正在接收时，LTSSM从L0切换到Recovery state，那么发生于LTSSM transition之后的Link Error能够被report
使用1b/1b encoding时，除了上图列出的那些states外，Link和Lane numbers不会在TS1/TS2 OS中发送
# Detect
detect阶段的子状态机如图所示：
detect阶段的主要目的是在电气上检测link的对端有没有device存在。同时，detect阶段也可以从LTSSM的其他阶段进入。
## Detect.Quiet
Detect.Quiet阶段是reset之后（Function Level Reset除外） / power-up之后的初始阶段，且必须在reset之后20ms内进入。同时，当其他阶段无法继续前进的时候，也可以进入该阶段。
该state的特性如下：
- Transmitter处于Electrical Idle state
- DC common mode voltage无需在SPEC规定的范围内
- intended data rate设置为2.5 GT/s。如果当进入该state时，intended data rate被设置为其他值，则LTSSM必须在该state停留至少1ms，且在这1ms内将intended data rate切换为2.5 GT/s。
- 注意，这个不影响后来使用TS1 / TS2 OS advertise的data rates，这个只是用于第一次训练到2.5 GT/s下的L0而已
- 对于所有的receiver，它必须在进入该substate的1ms内，满足2.5 GT/s下的。直到Z被满足，LTSSM必须停留在该substate内。
- 采用Physical Layer's status bit，即LinkUp=0b来通知DL层，link未处于opertional状态。即将LinkUp bit清零
- 注意，LinkUp status bit是一个内部的state bit，并不存在于configuration space中。该bit指示PL层何时完成链路训练，从而通知DL层和Flow Control初始化机制，让他们开始自己的link初始化。
- 清除任何以前的链路均衡成功状态。即清除Link Status 2 Register，16.0 GT/s Status Register，32.0 GT/s Status Register，64.0 GT/s Status Register中的phase 1 / phase 2 / phase 3 / equalization complete
- 以下field必须被清零，注意，以下的field有的也是硬件内部的register，并不通过PCIe规定的标准register暴露给software：
- 将use_modified_TS1_TS2_Ordered_Set重置为0，如果后来的训练真的要用到modified TS1 / TS2 OS，则这个bit会在configuration.LinkWidth.Accept state及以后发送的TS1 / TS2 OS中被set
- Device Status 3 Register中的Remote L0p Supported bit重置为0
- Flit_Mode_Enabled重置为0，如果链路真的要被训练为Flit Mode，则该bit会在configuration.LinkWidth.Accept state及以后发送的TS1 / TS2 OS中被set
- L0p_capable重置为0，该bit的值在Configuration.Complete state被决定
- SRIS_Mode_Enabled重置为0，该bit的值在Configuration.Lanenum.Accept state被决定
- 在Flit Mode下，当SRIS_Mode_Enabled为1时，SKP OS的发送频率必须遵循SRIS模式的频率。否则，遵循SRNS / Common Clock模式下要求的频率。其实就是SRIS下Tx和Rx端拥有更大的时钟偏差，因此发送SKP OS的频率要更大
- Directed_speed_change variable重置为0b. Upconfigure_capable variable重置为0b. Idle_to_rlock_transitioned variable重置为00h. Select_deemphasis variable重置为0b or 1b. based on platform specific needs for an Upstream Port. identical to the Selectable Preset/De-emphasis bit in the Link Control 2 Register for a Downstream Port. Equalization_done_8GT_data_rate, equalization_done_16GT_data_rate, equalization_done_32GT_data_rate, and equalization_done_64GT_data_rate variables重置为0b. Perform_equalization_for_loopback and perform_equalization_for_loopback_64GT variables重置为0b.
- 在12ms的超时之后，或者任意lane监测到Electrical Idle Exit之后，进入Detect.Active。因此，如果在任意lane上直接监测到Electrical Idle Exit，那么无需等12ms，直接进入Detect.Active即可。
- 如果LTSSM是从Hot Reset来进入该state的，则强烈建议Downstream Port在进入该state后，先等待2ms，而后开始Electrical Idle Exit的检测。
- 总的来说，Detect.Quiet state中，完成了内部register的清零和初始化
## Detect.Active
该state从Detect.Quiet substate进入。在该阶段下，Transmitter测试receiver有没有被连接到lane上。方法是首先设置一个任意的合法的DC common voltage，而后改变它。Detection Logic检查电压改变的速率，同时检测line上充放电的时间，将其与预期值比较，从而检测出link对端有没有device，如果有的话，充电时间会大大加长。（我们称之为电阻检测）
- 如果在所有的unconfigured lanes上都检测到receiver，从进入Polling。
- 如果在receiver处没有检测到任意lane，而返回Detect.Quiet。这两个substate的循环每12ms执行一次
- 如果有一部分，而非全部lane上检测到receiver，则
1. 等待12ms
2. 再次检测。
- 再次检测时，同样的一些lane上检测到receiver，而其他lane上没有，则进入Polling阶段。
- 对于那些没有detect到receiver的lanes
- 如果这些lanes能形成一个单独的link，则使用另一个LTSSM来管理它们，而后让那个LTSSM执行detect流，这个特性是optional的
- 如果不能使用另一个LTSSM，则没有检测到receiver的lanes将不会是link的一部分，而后进入Electrical Idle
- 当这些lanes进入Electrical Idle之前，无需发送一个EIOS
- 当目前的LTSSM重新回到Detect阶段之后，这些unconfigured lanes必须立即重新与LTSSM关联上
- 这种情况适用于link两端宽度不一样的情况，例如，一个x8宽度的RC接到了x4宽度的EP上
- 否则，进入Detect.Quiet
# Polling
Polling状态的子状态机如图所示：
在Polling阶段，link两端的device交换LTSSM TS1 / TS2。该阶段的主要目的在于让device理解link对端的传输内容。即：
- 建立起bit lock和symbol lock
- bit lock指的是Rx PLL成功从接收到的数据流中恢复出了clock，即Tx和Rx达成了时钟锁定
- symbol lock指的是Rx侧找到了接收的10-bit symbol的边界。这是通过COM symbol来实现的（因为Polling state只能在首次训练时，由detect进入，因此在Polling state中进行的bit lock和symbol lock使用的是8b/10b编码下的方法，注意和Recovery.RcvrLock中的bit lock/symbol lock过程区分）
- 解决极性反转
一旦这些过程完成，在每个device都能成功从link对端接收和发送TS Ordered Sets。
## Polling.Active
- Transmitter在所有detect state中探测到receiver的lanes上发送TS1 OS，其中Lane和Link numbers设置为PAD
- TS1 OS中的Data Rate Indentifier Symbol必须advertise device支持的在2.5 GT/s ～ 32.0 GT/s间的所有速率，即使device并不打算使用它
- 注意，在该state下，不允许advertise超过32.0 GT/s的data rates。即使该device支持64.0 GT/s速率，因为在此时link两端还没有协商好是否启用Flit Mode，64.0 GT/s及以上速率必须使用Flit Mode
- 对于Transmitter，在退出Electrical Idle，并且发送TS1 OS之前，Transmitter必须等待TX common mode被设定好
- 在进入该state的192 ns内，transmitter必须使用transmit margin field中的default voltage level来drive patterns。这个transmit voltage level必须持续有效，直到LTSSM进入Polling.Compliance或者Recovery.RcvrLock，在这2个state中，能够重新设置voltage level
- 如果Link Control 2 register中的Enter Compliance bit设置为1，则进入Polling.Compliance。如果这个bit在进入Polling.Active之前就被设置为1，则进入Polling.Active之后，必须立即切换到Polling.Compliance，即在Polling.Active阶段没有TS1被发送出去
- 在发送了至少1024个TS1s之后，所有在detect state中检测到receiver的lanes都接收到8个连续的Training Sequences（或者它们的每bit取反，这是由于极性反转引起的），且满足以下任意条件，则进入Polling.Configuration
- TS1s，且link / lane number为PAD，Compliance Receive bit(bit 4 of symbol 5)为0b
- TS1s，且link / lane number为PAD，Loopback bit(bit 2 of symbol 5)为1b
- TS2s，且link / lane number为PAD （这个条件是为了后离开Polling.Active的那个link partner使用的）
- 如果以上条件没有满足，则在24ms的timeout之后
- 满足以下所有条件，则进入Polling.Configuration（这种情况对应于有的lane损坏，LTSSM采用其他合法width训练好一个link）：
- 任意一个在detect state中检测到receiver的lane，在接收到TS1之后至少发送出去了1024个TS1s，并且接收到了8个连续的TS1 / TS2 OS（或者取反，这是对应电平反转的情况），这些OS的link / lane number设置为PAD，且满足以下任意条件（和上面那些常规训练流程中一致)
- TS1s，且link / lane number为PAD，Compliance Receive bit为0b
- TS1s，且link / lane number为PAD，Loopback bit为1b
- TS2s，且link / lane number为PAD
- 所有在detect state中检测到receiver的lane，至少有一个预先定义好的lane组，如x1 / x4等等，在进入Polling.Active阶段之后，已经检测到一次exit from Electrical Idle了
- 这防止了一个或者多个bad transmitter / receiver阻止valid link的configuration。
- 注意，前述的接收到8个连续的TS1/TS2 OS的lane，自从离开Polling.Active之后，已经能够更至少监测到一次exit from Electrical Idle。其实就是这条lane必须在这个lane组里
- Notes：在SNPS IP中，提供了开关，可以预设predetermined set of lanes为x1，或者所有在detect state中检测到receiver的lanes（在这种情况下，无法避免因为bad receivers / transmitters引起的无法训练的问题）
- 否则，如果以下任意条件满足则切换到Polling.Compliance
- 不是所有predetermined lanes，自从进入Polling.Active之后，都退出过Electrical Idle状态。这预示着有一个被动的测试负载，如电阻，连接到至少一个lane上，从而强制所有lanes进入Polling.Compliance，在SNPS IP中，如果假设predetermined lanes为所有在detect state中检测到receiver的lanes，则bad receivers / transmitters有可能会被误认为是被动测试负载，从而使得LTSSM错误地进入Polling.Compliance
- 任何一个在detect state中检测到receiver的lane收到8个连续的TS1s，其中link / lane number设置为PAD，Compliance Receive bit为1b，Loopback bit为0b
- 注意，如果port支持64.0 GT/s，则收到的TS1 OS中的Flit Mode Supported bit会被设置为1，Supported Link Speeds将为10111b
- 如果测试负载加到了所有lane上，则进入Polling.Compliance
- 如果在达到24ms的timeout时间之后，还是没有满足进入Polling.Configuration或者Polling.Compliance的条件，则LTSSM回到Detect state
## Polling.Compliance
- 在进入该substate时，采样Link Control 2 Register中的Transmit Margin field的值。而且采样得到的值，需要在进入该substate之后的192 ns内，在transmit package pins上有效，并且只要LTSSM还处在Polling.Compliance，则该margin field的值便一直有效
- 在该substate中发送compliance pattern所使用的data rate和de-emphasis level，在从Polling.Active到Polling.Compliance转变时便决定好，使用的算法如下所示
- 如果该Port仅仅支持以2.5 GT/s的速率发送数据，则发送Compliance Pattern的速率为2.5 GT/s，de-emphasis level为-3.5 dB（其实用的就是2.5 GT/s下规定的，简单的de-emphasis值）
- 否则，如果Port是因为在Polling.Active state中，检测到8个连续的TS1 OS，且Compliance Receive bit为1b，Loopback bit为0b，则Port在Polling.Compliance state中发送Compliance Pattern所使用的data rate如下所示：
- 如果该Port支持以64.0 GT/s的速率发送数据，并且，在Polling.Active state接收到的8个连续的TS1 OS中，Flit Mode Supported bit为1b，且Supported Link Speeds为1_0111b（这8个TS1 OS中，Compliance Receive bit为1b，Loopback bit为0b，即正是这8个TS1 OS将LTSSM从Polling.Active导向了Polling.Compliance），则该Port在Polling.Compliance中，以64.0 GT/s的速率发送Compliace Pattern。
- 否则，该Port发送数据所使用的速率，由Polling.Active state中，接收到的8个TS1 OS中的Data Rate Identifier中所advertise的最高速率决定
- 在发送compliance pattern时使用的链路均衡参数如下所示：
- 如果common data rate为8.0 GT/s或者更高速率，一条没有接收到8个连续的EQ TS1 OS的lane，或者接收到8个连续的EQ TS OS，但其中的Transmitter preset information是rsvd的lane，允许使用任何支持的transmitter preset value。
- 如果Port两端的common data rate为8.0 GT/s或者更高速率，则每条lane上，select_preset的值被设置为对应lane上收到的8个连续的EQ TS1 OS中的Transmitter Preset bits的值（前提是这一组链路均衡preset的值是合法的），并且这个值将会被Port的transmitter所使用（对于8.0 GT/s data rate，是否使用这些EQ TS1 OS中advertise的Receiver preset hint value是可选的）。
- select_deemphasis的值（该寄存器为硬件内部的寄存器值），必须被设置为在Polling.Active中接收到的8个连续的TS1 OS中的Selectable De-emphasis bit的值。
- Note: select_deemphasis这个值用于5.0 GT/s下的传输，当其为1b，则Tx使用-3.5 dB的de-emphasis，当其为0b，则Tx使用-6 dB的de-emphasis
- 否则，如果该Port的Link Control 2 Register中的Enter Compliace bit为1b（这意味有可能该Port在Polling.Active state中，根本就没有发送和接收TS1 OS，而是直接进入Polling.Compliace），则发送Compliace Pattern的速率，由Link Control 2 Register中的Target Link Speed field决定。如果用来发送Compliace Pattern的速率为5.0 GT/s，则当Link Control 2 Register中的Compliace Preset / De-emphasis field为0001b时，将内部的select_deemphasis设置为1b。如果用来发送Compliace Pattern的速率为8.0 GT/s或者更高，则每条lane上的select_preset variable必须被设置为Link Control 2 Register的Compliance Preset / De-emphasis value所提供的值，并且Tx也使用这个参数来发送Compliace Pattern（只要这个值不是rsvd）。（这段的意思在于，如果一个Port是因为自己的Link Control 2 Register的Enter Compliance bit为1b而主动进入Polling.Compliance的，则其工作速率 / Tx de-emphasis的值，都应该使用自己的Link Control 2 Register中的值）
- 否则，data rate，preset，de-emphasis的level settings，都按照如下表来定义。基于该Component的最大支持速率和因为这种原因进入Polling.Compliance的次数来计算。（该state适用于port连接到被动的测试负载上，port对方并不会发送任何TS OS的情况）（其实就是根据进入Polling.Compliance的次数，来一次次尝试data rate和均衡参数，直到能正常工作为止）
- 后续进入Polling.Compliace的次数，重复上述sequence。
- 如果Port支持16.0 GT/s或者更高速率，则在Polling.Configuration state中，必须将sequence设置为Setting #1。
- 如果Port工作于8.0 GT/s或者更高速率时，且其receivers不满足2.5 GT/s下的Zrx-dc，则必须将sequence设置为Setting #1
- 在Polling.Compliace State，所有的Ports都允许将sequence设置为Setting #1
- 上述切换流程的原理：测试负载板，可能会在任意一条lane的差分输出对的任意一条leg上，以100MHz的频率，350mV的幅值，发送持续大约1ms的信号，从而将device循环到想要的速率和de-emphasis level。因此，device必须要能够基于它的最大支持速率，来在表中的设置中按顺序循环，每次从Polling.Active进入Polling.Compliace时，都更新一次设置，整个循环从表中的第一行开始。
- 如果Compliance pattern data rate不是2.5 GT/s，并且进入Polling.Compliance之前的Polling.Active state中发送过任意TS 1 OS
- 如果以下条件均满足，则transmitter发送2个Compliace TS1 OS，否则不用发送
- 当前速率为2.5 GT/s
- Flit Mode is supported
- Transmitter正在按照上表中的设置进行循环
- 而后，transmitter发送一个EIOS，或者2个连续的EIOS，而后进入Electrical Idle
- 本段描述因为Link Control 2 Register中的Enter Compliance bit为1b而进入Polling.Compliance后的行为。如果compliance pattern date不是2.5 GT/s，并且在进入Polling.Compliance之前的Polling.Active state中没有发送任何TS1 OS出去，则在进入Polling.Compliace之后，Transmitter不发送任何EIOS，而是直接进入Electrical Idle。在Electrical Idle中，将速率切换到新的速率并且稳定下来。如果将要工作的速率为5.0 GT/s，则如果select_deemphasis value为1b，de_emphasis / preset level为-3.5dB，否则为-6dB。如果将要工作的速率为8.0 GT/s或者更快，则Transmitter preset value必须被设置为select_preset的值。（其实就是按照Link Contrl 2 Register中的值来配置data rate和均衡参数，因为这些内部寄存器的值都是在从Polling.Active转变为Polling.Compliace时，根据Link Control 2 Register中的值来配置的）注意Electrical Idle的时间大于1ms而小于2ms。
- 在data rate和de-emphasis / preset level被决定好之后，Polling.Compliance中的行为如下所示：
- 如果Port是因为如下原因之一进入Polling.Compliance，则port的transmitter以上述决定好的data rate，在所有在detect state中探测到receiver的lanes上，发送modified compliance pattern，且error status symbol被设置为全0
- 在Polling.Active中，探测到8个连续的TS1 OS，且其中Compliance Receive bit为1b，而Loopback bit为0b
- Link Control 2 Register中Enter Compliance bit和Enter Modified Compliance bit都为1b
- 如果发送Compliance Pattern的data rate为2.5 GT/s或者5.0 GT/s，
- 如果发送Compliance Pattern的data rate为8.0 GT/s或者更高
- Non-Flit Mode和Flit Mode下定义的Scrambling requirements适用于received Modified Compliance Pattern。例如，scrambling LFSR的seed是按照每条lane单独进行配置的，EIEOS初始化link两端的LSFR，且SKP OS不会使得LFSR shift
- Handling bit slip and block alignment：device必须确保他们的reciever已经稳定下来了，而后才能尝试获得Block Alignment，并且宣告Pattern Lock。例如，如果一个implementation在最初的一些bits中会看见bit偏移，则它需要等待偏移时间结束，而后再设置Block Alignment。Device也有可能想要在设置Pattern Lock bit之前，重新配置它的Block Alignment
- 如果data rate为2.5 GT/s或者5.0 GT/s，一旦某个lane指示它已经lock到接收的Modified Compliance Pattern上了，则每次Receiver Error发生的时候，那个Lane的Receiver Error Count都会加1
- error status symbol使用lower 7 bits作为Receiver Error Count field，并且当计数满127时，会一直停留在全1的状态
- 如果使用8b/10b编码，receiver不允许对它将要接收的10-bit pattern作任何假设
- 如果Link Control 2 Register中的Enter Compliance bit为0b，则如果上层有引导，则进入Detect。（其实意思就是在Link Control 2 Register中的Enter Compliance bit为0b的情况下，才能将LTSSM引导进入Detect）
- 否则，如果在进入Polling.Compliance时，Enter Compliance bit为1b，则如果下列任意情形之一满足，进入Polling.Active
- Link Control 2 Register中的Enter Compliance bit变为0b
- 该Port为Upstream Port，且任意lane上接收到一个EIOS。当该条件满足时，将Enter Compliance bit清零
- 如果transmitter正在以2.5 GT/s之外的速率发送compliance pattern，或者在进入Polling.Compliance的过程中，link Contol 2 Register中的Enter Compliance bit被设置为1b，则transmitter在切换到Polling.Active之前，要首先发送8个连续的EIOS，并且进入Electricla Idle。在Electrical Idle中，需要将data rate切换到2.5 GT/s并且稳定。并且de-emphasis level会被重置为-3.5dB。注意在Electrical Idle的时间必须大于1ms，但不允许超过2ms。Note：发送多个EIOS，从而能够提供足够的健壮性，使得其他port至少能检测到一个EIOS，并且退出Polling.Compliance substate。
- 否则，如果Port是因为Link Control 2 Register中的Enter Compliance bit为1b，且Enter Modified Compliance bit为0b，而进入Polling.Compliance的
- Transmitter在所有在detect state中检测到receiver的lane上发送Compliance pattern，速率和de-emphasis / preset level的选择前文已经有述
- 如果下列情形之一成立，则进入Polling.Active
- 进入Polling.Compliance之后，Link Control 2 Register中的Enter Compliance bit从1b变成了0b
- Port是Upstream Port，Link Control 2 Register中的Enter Compliance bit为1b，但在任意lane上接收到一个EIOS。当该条件满足时，将Enter Compliance bit清零
- 在Electrical Idle中，需要将data rate切换到2.5 GT/s并且稳定。并且de-emphasis level会被重置为-3.5dB。注意在Electrical Idle的时间必须大于1ms，但不允许超过2ms。Note：发送多个EIOS，从而能够提供足够的健壮性，使得其他port至少能检测到一个EIOS，并且退出Polling.Compliance substate。
- 否则
- a. transmitter在所有detect state中检测到receiver的lane上，发送下列patterns。使用的data rate和de-emphasis / preset level由前文所述决定。
- 如果在任意一个在Detect state中检测到receiver的lane上，它的receiver检测到exit to Electrical Idle，则进入Polling.Active。如果transmitter正在以2.5 GT/s之外的速率发送data，则在转为Polling.Active之前，transmitter发送8个连续的EIOS，并且进入Electrical Idle。在Electrical Idle阶段，data rate切换到2.5 GT/s并且稳定下来，Electrical Idle的时间必须大于1ms，但不允许超过2ms
## Polling.Configuration
- 在该substate中，如有需要，receiver必须反转接收到的bit流的极性。在此substate中配置极性反转，以便于后续Configuration State中TS OS的交换
- 当进入该substate时，Link Control 2 Register中的Transmit Margin Field必须被重置为000b
- 在该substate中，transmitter将会在detect state检测到receiver的所有lanes上，停止发送TS1s，转而发送TS2s，注意在TS2s中，link / lane number field仍然是PAD
- TS2 OS中的Data Rate Indentifier Symbol必须advertise device支持的在2.5 GT/s ～ 32.0 GT/s间的所有速率，即使device并不打算使用它
- 注意，在该state下，不允许advertise超过32.0 GT/s的data rates。即使该device支持64.0 GT/s速率。因为Link两端还没有协商Flit Mode support
- 发送TS2s的目的是为了告诉link partner，自己的状态机已经准备进入下一个state了。这是一种握手机制，用于保证Link两端的device的LTSSM进展一致，即直到link两端都准备好之后，一侧的LTSSM才允许进入下一个state。当一个device同时发送和接收TS2s时，它知晓自己已经能够进入下一个state了（因为自己ready了，且自己的link partner也ready了）
- 先进入Polling.Configuration的link partner先发送TS2 OS，而后接收TS2 OS。而后进入Polling.Configuration的link partner先因为接收到TS2 OS进入Polling.Configuration，而后再发送TS 2 OS。
- 在detect阶段中检测到receiver的任意lane上，接收到了8个连续的TS2s，且link / lane number为PAD。并且在接收到一个TS2之后，已经发送出去了16个TS2，则进入Configuration State
- 否则，在48ms的timeout之后，返回detect state
## Polling.Speed
该substate已经弃用且无法到达。因为Link会在2.5 GT/s的速率下训练到L0状态，而后通过进入Recovery状态来切换速率
即使Link能够在2.5 GT/s以上的速率中工作，一个link最初将会在2.5 GT/s的速率下训练至L0。而受支持的更高速率使用TS Ordered Sets进行advertise。link对端支持的速率将会在Configuration.Complete状态中被寄存下来。基于link两端所共同支持的最高速率，link的任意一端都允许通过从L0切换到Recovery来发起一次速率切换。
# Configuration
configuration的子状态机如图所示：
最初，Configuration state在2.5 GT/s下执行link和lane number的编号。但是，后来允许更高的速率的device由recovery阶段进入configuration，从recovery进入configuration的主要目的是改变link的宽度。但是在Gen6及以上的PCIe协议中，又引入了新的L0p substate，取代了进入Configuration State的切lane方法
Configuration阶段的主要目的是决定port是如何连接的，并且为link分配lane number。Leader，即Downstream Port，将向Upstream Port描述link和lane numbers。在没有冲突的情况下，Upstream port返回同样的值。
它的一些substates本质上就是一个反复握手的过程。
## Configuration.Linkwidth.Start
该substate有两种方法进入：
- 完成Polling state，而后进入，这是常规的训练流程
- 在recovery state，发现从上一次分配之后， link / lane number已经发生了改变，因此recovery过程不能正常结束，需要进入Configuration阶段重现分配link / lane numbe
### Downstream Lanes
- 如果Downstream Port的上层指示，将所有在Detect阶段检测到Receiver的lane上发送的TS1 / TS2的Disable Link bit置为1，则将其引导进入Disabled
- 如果Downstream Port的上层指示，将所有在Detect阶段检测到Receiver的lane上发送的TS1 / TS2的Loopback bit置为1，且transmitter有能力作为一个Loopback Leader，则将其引导进入Loopback
- 如果以下任意情形存在，则将Downstream Port引导进入Loopback
- 所有正在发送和接收TS1 OS的lanes上，都接收到了两个连续的TS1 OS，且其中的Loopback bit设置为1
- 任意正在发送TS1 OS的lane上，接收到了两个连续的TS1 OS，且其中的Loopback bit设置为1，Enhanced Link Behavior Control bits设置为01b（这也是TS1 OS的一个bit）
- 注意，如果一个Port支持以64.0 GT/s的速率传输，则它接收到的TS1 OS中，可能Flit Mode Supported bit为1，且Supported Link Speeds field为1_0111b。
- 接收到Ordered Sets，且其中的Loopback bit为1的port，则成为Loopback Follower。如果Loopback Follower支持1b/1b编码，则它必须考虑是否启用了SRIS clocking，这用于在Loopback中，辨别SKP OS的边界。
- 如果LinkUp=0b，或者如果LTSSM没有正在发起升Lane过程，则Transmitter在所有的active Downstream lanes上发送TS1 OS，且其中的Link number不为PAD，而Lane number为PAD。（其实就是首次训练，分配link number的过程）。此外，如果Upconfigure_capable为1b，且LTSSM没有在发起升lane过程，则Transmitter也在下列inactive lanes上面发送TS1 OS，且其中的Link number不为PAD，而Lane number为PAD。这些inactive lanes如下（这对应允许升lane的LTSSM）。Port在这些inactive lanes上，自从进入Recovery之后，已经detect到了exit from Electrical Idle，并且之后，在该state中，接收到了2个连续的TS1 OS，且Link / Lane number都为PAD
- 当从polling state切换到该substate时，任意在Detect阶段检测到Receiver的lane，都被视为active lane
- 当从Recovery state切换到该substate时，上一次通过Configuration.Complete配置好的Link上的所有lanes，都被视为active lane
- TS1 OS的Data Rate Identifier Symbol必须advertise该Port支持的所有data rates，即使该port不打算使用该data rate
- 如果LinkUp=1b，且LTSSM正在发起升Lane的过程，则首先，在下列lanes上发送TS1 OS，且Link number和Lane number都为PAD
- 现在active的lanes
- 它想要active的inactive lanes
- 那些自从进入Recovery之后，已经detect到了exit from Electrical Idle，并且已经接收到了2个连续的TS1 OS，且Link和Lane Number都为PAD的lanes
- 当上述的所有发送TS1 OS的lanes上，都接收到了2个连续的TS1 OS，且其Link number和Lane number都为PAD，或者直接在进入该state 1ms之后，LTSSM开始在这些lanes上发送TS1 OS，且其中的Link number不为PAD，而Lane number为PAD
- 在启用了任意一个inactive lane之后，Transmitter必须等待它的TX common mode设置好，而后才从Electrical Idle中退出，并且发送TS1 OS
- 只有在形成不同link的lane上，才允许使用不同的link number
- Note：一个例子是8条Downstream Lanes，能够协商为一个x8 port并且连接到1个component，或者协商为2个x4 port，并且连接到2个不同的component。在协商为2个x4 ports时，对于4条lanes，Downsteam Lanes发送TS1 OS，且Link Number为N，而另外4条Lanes，Link Number为N+1。注意Lane Number都为PAD。
- 对于Downstream Port而言，如果任何Lanes首先接收到至少1个或者多个TS1 OS，且Link / Lane number为PAD，而后在这些Downstream Lanes上，有任意一个Lanes接收到2个连续的TS1 OS，且Link number不为PAD（而是和Downstream Port先前发送出去的TS1 OS中的Link Number一致），而Lane number为PAD，则进入Configuration.Linkwidth.Accept。（其实这就是握手完成，就是Downstream Port先发送Link number不为PAD而Lane number为PAD的TS1 OS，而后当它接收到2个Upstream Port返回来的，包含相同Link Number的TS1 OS，则握手完成，自己进入Configuration.Linkwidth.Accept）
- 在24ms的timeout之后，进入Detect
### Upstream Lanes
- 如果Port的上层将所有在Detect阶段检测到Receiver的lane上发送的TS1 / TS2的Loopback bit置为1，则将其引导进入Loopback
- 如果任意正在发送TS1 OS的lanes上接收到了2个连续的TS1 OS，且其中的Disable Link bit为1，则进入Disabled（其实就是对面的Downstream Port被它的upper layer引导向Disabled了，则Downstream Port发送TS1 OS，且Disable Link bit为1出去）
- 如果以下任意条件满足，则进入Loopback（其实就是对面的Downstream Port被它的upper layer引导向Loopback了，则Downstream Port发送TS1 OS，且Loopback bit为1b出去）：
- 所有正在发送和接收TS1 OS的lanes上，都接收到了两个连续的TS1 OS，且其中的Loopback bit设置为1
- 任意正在发送TS1 OS的lane上，接收到了两个连续的TS1 OS，且其中的Loopback bit设置为1，Enhanced Link Behavior Control bits设置为01b
注意，如果一个Port支持以64.0 GT/s的速率传输，则它接收到的TS1 OS中，可能Flit Mode Supported bit为1，且Supported Link Speeds field为1_0111b。
接收到Ordered Sets，且其中的Loopback bit为1的port，则成为Loopback Follower。如果Loopback Follower支持1b/1b编码，则它必须考虑是否启用了SRIS clocking，这用于在Loopback中，辨别SKP OS的边界。
- 首先，Upstream port的transmitter在下列所有lane上发送TS1 OS，且Link number / Lane number为PAD
- 所有active lane
- 所有预备升lane的inactive lanes
- 如果Upconfigure_capable为1，那些自从进入Recovery后探测到Exit from Electrical Idle，且已经检测到两个连续的TS1 OS（Link/Lane number为PAD）的lanes
- 当从polling state切换到该substate时，任意在Detect阶段检测到Receiver的lane，都被视为active lane
- 当从Recovery state切换到该substate时，上一次通过Configuration.Complete配置好的Link上的所有lanes，都被视为active lane
- 当从Recovery state切换到该substate时，如果transaction不是由于LTSSM timeout引起的，且transmitter由于autonomous reasons想要改变link width，则transmitter必须将自己在Configuration state中发送的TS1 OS中的Autonomous Change bit(symbol 4 bit 6)设置为1b。这指示该此宽度切换是由于功耗原因，而非工作可靠性原因
- TS1 OS的Data Rate Identifier Symbol必须advertise该Port支持的所有data rates，即使该port不打算使用该data rate
- 而后，如果任意lane接收到2个连续的TS1 OS，且link number不为PAD，而lane number为PAD，则进入Configuration.LinkWidth.Accept，而后在所有在detect阶段检测到receiver，且接收到与上述一致的两个TS1 OS的lane上，发送TS 1 OS，且link number不为PAD，而Lane number为PAD。而剩下的在detect阶段检测到receiver的lanes（但是没有接收到与上述一致的两个TS1 OS），则发送TS1 OS，且link / lane number都为PAD。（其实就是接收到2个连续的TS1 OS之后，且Link number不为PAD之后，Upstream Port就进入Configuration.LinkWidth.Accept了，而后开始向上发送link number不为PAD的TS1 OS，换言之，在这种情况下，Upstream Port比Downstream Port更先进入Configuration.LinkWidth.Accept）
- 如果LTSSM正在发起升lane，则它会一直等待下列情况之一发生，而后发送TS1 OS，且且link number不为PAD，而Lane number为PAD。直到所有它想要active的inactive lanes上都接收到了2个连续的TS1 OS，且Link number不为PAD而Lane number为PAD。或者在进入该substate 1ms后，在任意它想要active的inactive lanes上接收到了2个连续的TS1 OS，且Link number不为PAD而Lane number为PAD
- 建议任何接收到一个错误的TS1 OS，或者丢失对齐的multi-lane link（比如丢失了128b/130b Block Alignment，或者丢失了1b/1b Block/Flit Alignment），延迟一段时间再进行上述的判断（在8b/10b编码下，延迟2个或者更多的TS1 OS，在128b/130b或者1b/1b编码中，延迟34个或者更多的TS1 OS），但是总延迟时间不允许超过1ms。从而防止构建一个比实际需要更小的link
- 在activating任意一个inactive lane之后，Transmitter必须等待它的TX common mode设置好，而后才从Electrical Idle中退出，并且发送TS1 OS
- 只有在形成不同link的lane上，才允许使用不同的link number
- 在24ms的timeout之后，进入Detect
## Configuration.LinkWidth.Accept
在该substate，Upstream Port发回TS1 OS，其中link number与其收到的一致。现在Downstream知道具体的link width了，因此，它继续发送TS1 OS，但是其中的Lane number不再是PAD，而是具体的Lane numbers
### Downstream Lanes
- 如果至少能用接收到2个连续的TS1 OS（这些TS1 OS中Link number为non-PAD，且与Downstream Port在Configuration.LinkWidth.Start中发送的TS1 OS中的Link number一致）的lanes中的一部分lane来形成一个link，则开始发送TS1 OS，使用同样的Link Number，以及non-PAD的lane number，而后进入Configuration.Lanenum.Wait
- 分配的non-PAD lane number的范围为0～n-1，在使用同一个Link number的lane上顺序分配。而剩下的未使用的lane上继续使用Link / Lane number PAD
- 建议任何接收到一个错误的TS1 OS，或者丢失对齐的multi-lane link（比如丢失了128b/130b Block Alignment，或者丢失了1b/1b Block/Flit Alignment），延迟一段时间再进行上述的判断（在8b/10b编码下，延迟2个或者更多的TS1 OS，在128b/130b或者1b/1b编码中，延迟34个或者更多的TS1 OS），但是总延迟时间不允许超过1ms。从而防止构建一个比实际需要更小的link
- 如果下列所有的情况都满足，则必须将use_modified_TS1_TS2_Ordered_Set设置为1b，这意味这后续发送的TS1/TS2 OS都变成了Modified TS1/TS2 OS
- 自从进入Polling state之后，在Polling state和Configuration state中，已经发送过TS1 / TS2 OS，其中的Enhanced Link Behavior Control field中的Modified TS1 / TS2 OS supported value为11b
- LinkUp=0b，即还没有建立起连接
- 在当前configure的link的所有lane上，接收到的8个连续的TS2 OS（此处指的是导致LTSSM从Polling.Configuration切换到Configuration的那8个TS2 OS）中，Enhanced Link Behavior Control field中的Modified TS1/TS2 Ordered Sets supported为11b，且32.0 GT/s data rate is supported bit为1b
- 如果下列所有的情况都满足，则必须将Flit_Mode_Enable设置为1b
- LinkUp=0b
- 自从进入Polling state之后，在Polling state和Configuration state中，已经发送过TS1 / TS2 OS，其中的Data Rate Identifier field中的Flit Mode Supported bit为1
- 在当前configure的link的所有lane上，接收到的8个连续的TS2 OS（此处指的是导致LTSSM从Polling.Configuration切换到Configuration的那8个TS2 OS）中，Data Rate Identifier field(Symbol 4, Bit 0)中的Flit Mode Supported bit设置为1b
- 下列任意情况满足的情况下，进入Detect
- 2ms的timeout之后
- no link can be configured
- 所有的lanes都接收到2个连续的TS1 OS，且link / lane number为PAD
### Upstream Lanes
- 如果能够用一些lanes来形成一个link，则在这些lanes上发送TS1 OS，且Link number不为PAD，而Lane number和接收到的一致，或者顺序相反（考虑到lane reversal）。而后进入Configuration.Lanenum.Wait
- 接收到的TS1 OS可能是标准的TS1 OS，也有可能是Modified TS1 OS。只有当满足 set use_modified_TS1_TS2_Ordered_Set的情况下，才有可能接收到Modified TS1 OS
- 新分配的lane number范围为0～m-1。注意必须包含lane 0或者lane n-1。其他没有形成link的lanes，发送TS1 OS，且link / lane number为PAD
- 如果multi-lane link接收到了一个错误的TS1 OS，或者丢失了128b/130b block alignment，或者在1b/1b编码下丢失了Block/Flit alignment，则建议执行以下操作，来防止配置出一个比需要更窄的link
- 当使用8b/10b编码时，延迟评估2个或者更多个TS1 OS，但是不允许延迟超过1ms
- 当使用128b/130b或者1b/1b编码时，延迟评估34个或者更多TS1 OS，但是不允许延迟超过1ms
- 如果下列所有条件为真，则将use_modified_TS1_TS2_Ordered_Set设置为1b
- 与Downstream Port中的一致
- 如果下列所有条件为真，则将Flit_Mode_Enable设置为1b
- 与Downstream Port中的一致
- 下列任意情况满足的情况下，进入Detect
- 2ms的timeout之后
- no link can be configured
- 所有的lanes都接收到2个连续的TS1 OS，且link / lane number为PAD
## Configuration.Lanenum.Wait
该substate存在的意义：对于一个Multi Lane Link，如果在Configuration过程中，在有的Lane上探测到了错误，或者丢失了Block对齐，或者由于line skew的原因，需要等待一段时间来尝试恢复，而不是直接不配置这条lane了（不配置这条lane会导致配置出来的Link宽度比实际需要的小）
在该sub-state下，如果use_modified_TS1_TS2_Ordered_Set设置为1b，则：
- Transmitter必须发送Modified TS1 OS，而不是普通的TS1 OS
- Receiver必须检测Modified TS1 OS，而不是普通的TS1 OS。但是当Link Partner正在切换到该sub-states，Receiver仍然有可能接收到TS1 OS。
### Downstream Lanes
- 如果任意一条在detect state中检测到receiver的lane上，接收到了2个连续的TS1 OS，其中Lane number和这条lane首次进入Configuration.Lanenum.Wait时的不一致，并且并非所有lane的link number都被设置为PAD。或者在所有lanes上接收到了2个连续的TS1 OS，且link和lane number和正在发送的一致，则进入Configuration.Lanenum.Accept。
- 在切换到Configuration.Lanenum.Accept之前，允许Upstream Lanes等待最少1 ms
- 在切换state之前，等待1 ms的理由包括防止receiver errors或者lanes之间的skew，以防影响到最后配置的Link的宽度
- 如果到了2ms的timeout，或者所有的lanes都接收到了两个连续的TS1 OS，且其中的Link / Lane number都为PAD，则进入到Detect
### Upstream Lanes
- 下列条件有一个满足，则进入Configuration.Lanenum.Accept
- 任何一条lane接收到了2个连续的TS1 OS，而且其中的Lane number和这条lane刚进入Configuration.Lanenum.Wait时不同，并且不是所有的lane上接收到的TS OS中的Link Number都为PAD
- 任意lane接收到了2个连续的TS2 OS
- 在2ms的timeout之后，或者所有的lanes都接收到了2个连续的TS1 OS，且link / lane number均为PAD，则进入Detect
## Configuration.Lanenum.Accept
在该sub-state中，如果use_modified_TS1_TS2_Ordered_Set为1b，则：
- Transmitter必须发送Modified TS1 OS，而不是普通的TS1 OS
- Receiver必须检测Modified TS1 OS，而不是普通的TS1 OS。但是当Link Partner正在切换到该sub-states，Receiver仍然有可能接收到TS1 OS
### Downstream Lanes
- 如果接收到了2个连续的TS1 OS，且Link number和自己在Configuration.LinkWidth.Accept中发出去的一样，而lane number和发出去的一样或者相反（考虑到lane reversal），则进入Configuration.Complete。如果use_modified_TS1_TS2_Ordered_Set为1，而且正在执行Alternate Protocol协商，则进入Configuration.Complete必须被延迟10us，或者直到Downstream port接收到Upstream port对protocol request的response。
- 如果下列所有情形都满足，则将SRIS_Mode_enabled设置为1
- LinkUp=0b
- 自从进入Configuration state之后，在传输的TS1 OS中，一直将SRIS Clocking bit设置为1b
- 如果LTSSM是从recovery阶段进入configuration阶段的，则当link宽度发生改变时，Link Status Register中的Link Bandwidth Management Status and Link Autonomous Bandwidth Status bits必须同时发生改变
- 如果是Downstream因为工作可靠性的原因发起的link宽度改变，则将Link Bandwidth Management Status bit设置为1
- 如果link宽度改变不是由Downstream port发起的，而且在接受到的两个连续的TS1 OS中，Autonomous Change bit为0b，则将Link Bandwidth Management Status bit设置为1
- 否则，将Link Autonomous Bandwidth Status bit设置为1
- Lane reversal严格定义为Downstream Port lane 0接收到TS1，其中lane number为n-1。且Downstream Port lane n-1接收到TS1，其中lane number为0
- 如果multi-lane link接收到了一个错误的TS1 OS，或者丢失了128b/130b block alignment，或者在1b/1b编码下丢失了Block/Flit alignment，则建议执行以下操作，来防止配置出一个比需要更窄的link
- 当使用8b/10b编码时，延迟评估2个或者更多个TS1 OS，但是不允许延迟超过1ms
- 当使用128b/130b或者1b/1b编码时，延迟评估34个或者更多TS1 OS，但是不允许延迟超过1ms
- 如果能够使用那些接收到2个连续的TS1 OS（拥有相同的non-PAD Link Numbers和任意Non-PAD Lane numbers）的lanes中的一部分来形成一个link，则在这些lanes上发送TS1 OS，其中有同样的non-PAD Link numbers和新的Lane Numbers。并且切换到Configuration.Lanenum.Wait
- 新分配的lane numbers范围必须是0- m-1，并且只能分配给连续的lanes。而且必须包含Lane0或者Lane n-1，且M-1必须小于等于N-1.任意剩下的lanes发送TS1 OS，其中Link和Lane numbers为PAD
- 如果multi-lane link接收到了一个错误的TS1 OS，或者丢失了128b/130b block alignment，或者在1b/1b编码下丢失了Block/Flit alignment，则建议执行以下操作，来防止配置出一个比需要更窄的link
- 当使用8b/10b编码时，延迟评估2个或者更多个TS1 OS，但是不允许延迟超过1ms
- 当使用128b/130b或者1b/1b编码时，延迟评估34个或者更多TS1 OS，但是不允许延迟超过1ms
- 如果没有不能构建任何Link，或者所有的lanes都接收到2个连续的TS1 OS，且Link / Lane number为PAD，则进入Detect
### Upstream Lanes
- 如果接收到2个连续的TS2 OS，且其中Link / Lane number不为PAD，且与自己发送出去的TS1 OS中的link / lane number一致，则进入Configuration.Complete。如果use_modified_TS1_TS2_Ordered_Set设置为1，且正在执行Alternate Protocol Negotiation，并且Downstream Port决定不使用任何Alternate Protocol，则接收到的TS2 OS应该将Modified TS Usage设置为定义好的值。
- 如果以下条件满足，则将SRIS_Mode_Enabled bit设置为1
- LinkUp=0b
- 任意一条lane上接收到的2个连续的TS2 OS中，都将SRIS Clocking bit设置为1
- 如果能够使用那些接收到2个连续的TS1 OS（拥有相同的non-PAD Link Numbers和任意Non-PAD Lane numbers）的lanes中的一部分来形成一个link，则在这些lanes上发送TS1 OS，其中有同样的non-PAD Link numbers和新的Lane Numbers。并且切换到Configuration.Lanenum.Wait
- 新分配的lane numbers范围必须是0- m-1，并且只能分配给连续的lanes。而且必须包含Lane0或者Lane n-1，且M-1必须小于等于N-1.任意剩下的lanes发送TS1 OS，其中Link和Lane numbers为PAD
- 如果multi-lane link接收到了一个错误的TS1 OS，或者丢失了128b/130b block alignment，或者在1b/1b编码下丢失了Block/Flit alignment，则建议执行以下操作，来防止配置出一个比需要更窄的link
- 当使用8b/10b编码时，延迟评估2个或者更多个TS1 OS，但是不允许延迟超过1ms
- 当使用128b/130b或者1b/1b编码时，延迟评估34个或者更多TS1 OS，但是不允许延迟超过1ms
## Configuration.Complete
当进入该substate时，允许port改变它advertise的supported data rates，但是当它处于该substate时，就不允许改变了。
如果Flit_Mode_Enable为0b，且LinkUp为1b，则当它进入该substate时，允许他改变自己advertise的Upconfigure。同理，当他处于该substate时，就不允许改变了
如果Flit_Mode_Enabled为1b，且LinkUp为0b，则当它进入该substate时，允许改变它的L0p Capability。同理，当他处于该substate时，就不允许改变了
在该substate中，如果use_modified_TS1_TS2_Ordered_Set设置为1，则
- Transmitter必须发送Modified TS2 OS，而非普通的TS2 OS
- Receivers必须监控Modified TS2 OS，而非普通的TS2 OS
### Downstream Lanes
- 发送TS2 OS，其中Link / Lane number与接收到的TS1 OS中的Link / Lane number一致
- 允许在TS2 OS中，将Link Upconfigure / L0p Capability设置为1b，来指示该Port支持downsize to x1 link，且使用的是当前分配的Lane 0。而且当LinkUp=1b时，支持up-configuring。
- 当离开该substates，必须注意L0s中使用的N_FTS
- 当使用8b/10b编码时，当离开该substate时，必须做好Lane-to-Lane de-skew
- 如果所有的配置好的lanes上，都接收到了2个连续的TS2 OS，且其中的Disable Scrambling设置为1，则disable scrambling
- 在所有lanes上发送Disable Scrambling bit的port，也要disable scrambling。注意，只有在使用8b/10b编码时，才能disable scrambling
- 当所有正在发送TS2 OS的lanes，都接收到8个连续的TS2 OS，且Link / Lane number以及data rate identifier与发送出去的一致，并且在接收到1个TS2 OS之前自己已经发送出去了16个TS2 OS，则进入Configuration.Idle。当Link Capabilities 2 Register中的Retimer Presence Detect Supported bit或者Two Retimers Presence Detect Supported bit为1b时，当处于2.5 GT/s时，必须同时接收到8个连续的TS2 OS，且拥有同样的Retimer Preset bit。
- 如果工作的data rate为2.5 GT/s
- 如果Link Capabilities 2 Register中的Retimer Presence Detect Supported bit为1b，在任意lane上接收到8个连续的TS2 OS，且其中的Retimer Present bit为1b，则必须将自己的Link Status 2 Register中的Retimer Presence Detected bit设置为1b，否则将其设置为0b
- 如果Link Capabilities 2 Register中的Two Retimers Presence Detect Supported bit为1b，在任意lane上接收到8个连续的TS2 OS，且其中的Two Retimers Present bit为1b，则必须将自己的Link Status 2 Register中的Two Retimers Presence Detected bit设置为1b，否则将其设置为0b
- 如果Device支持大于2.5 GT/s的速率，则它必须记录下它在link的任意lane上接收到的data rate identifier。这将覆盖以前接收到的值。在recovery阶段追踪speed change的变量，即changed_speed_recovery，将被重置为0b
- 如果Flit_Mode_Enabled为0b。如果该device发送TS2 OS，且其中的Link Upconfigure / L0p Capability设置为1b，而且在接受到的8个连续的TS2 OS中，Link Upconfigure / L0p Capability同样为1b，则将upconfigure_capable设置为1b，否则将其重置为0b
- 如果Flit_Mode_Enabled为1b，且LinkUp为0b。如果该device发送TS2 OS，且其中的Link Upconfigure / L0p Capability设置为1b，而且在接受到的8个连续的TS2 OS中，Link Upconfigure / L0p Capability同样为1b
- L0p_capable设置为1b
- Device Status 3 Register中的Remote L0p Supported bit设置为1b
- 所有的剩下的不是Configured Link一部分的lanes，与LTSSM不再关联，而且必须执行以下操作
- 如果optional feature is supported，则将其与一个新的LTSSM相关联
- 所有的，不能与新的可选的LTSSM相关联的lanes都必须切换到Electrical Idle。
- 对于升/降lane的情况，如果LTSSM advertise Link width upconfigure capability，则那些在L0 state中形成了link，且LinkUp为1b lane，他们虽然不再是当前配置的link的一部分，但是不允许将其与LTSSM解耦。建议这些lanes的receiver保持开启状态。但是如果关闭了的话，则如果upconfigure_capable为1b，则当LTSSM进入Recovery.Rcvrcfg，直到LTSSM进入Configuration.Complete阶段，它们必须开启，以允许可能的升lane操作。在最初的link training to L0过程中，并没有成为LTSSM一部分的lanes，在升lane过程中，也不允许成为link的一部分
- 当LTSSM切换回Detect阶段后，这些lanes必须立即与LTSSM重新关联
- 在切换到Electrical Idle之前，无需发送EIOS，且向EIOS的切换也无需发生于Symbol / OS的边界处
- 在2ms的timeout之后
- 如果当前速率为2.5 GT/s或者5.0 GT/s，则切换到Detect
- 如果当前速率为8.0 GT/s或者更高，且idle_to_rlock_transitioned小于FFh，则切换到Configuration.Idle
- 将changed_speed_recovery重置为0b
- 那些不是当前Configured Link的lanes，不再与LTSSM相关联
- 如果至少有一条lane接收到了8个连续的TS2 OS，且拥有符合的Link / L按哦number，则upconfigure_capable允许，但是不一定要更新。如果更新，则如果发送和接收的Link Upconfigure为1b，则将upconfigure_capable设置为1b，否则设置为0b
- 否则，切换到Detect
### Upstream Lanes
- 发送TS2 OS，并且使用的Link / Lane number和接收到的TS2 OS中的一致
- 允许在TS2 OS中，将Link Upconfigure / L0p Capability设置为1b，来指示该Port支持downsize to x1 link，且使用的是当前分配的Lane 0。而且当LinkUp=1b时，支持up-configuring
- 当离开该substates，必须注意L0s中使用的N_FTS
- 当使用8b/10b编码时，当离开该substate时，必须做好Lane-to-Lane de-skew
- 如果所有的配置好的lanes上，都接收到了2个连续的TS2 OS，且其中的Disable Scrambling设置为1，则disable scrambling
- 在所有lanes上发送Disable Scrambling bit的port，也要disable scrambling。注意，只有在使用8b/10b编码时，才能disable scrambling
- 当所有正在发送TS2 OS的lanes，都接收到8个连续的TS2 OS，且Link / Lane number以及data rate identifier与发送出去的一致，并且在接收到1个TS2 OS之后自己发送出去了16个TS2 OS，则进入Configuration.Idle。当Link Capabilities 2 Register中的Retimer Presence Detect Supported bit或者Two Retimers Presence Detect Supported bit为1b时，当处于2.5 GT/s时，必须同时接收到8个连续的TS2 OS，且拥有同样的Retimer Preset bit。
- 如果工作的data rate为2.5 GT/s
- 如果Link Capabilities 2 Register中的Retimer Presence Detect Supported bit为1b，在任意lane上接收到8个连续的TS2 OS，且其中的Retimer Present bit为1b，则必须将自己的Link Status 2 Register中的Retimer Presence Detected bit设置为1b，否则将其设置为0b
- 如果Link Capabilities 2 Register中的Two Retimers Presence Detect Supported bit为1b，在任意lane上接收到8个连续的TS2 OS，且其中的Two Retimers Present bit为1b，则必须将自己的Link Status 2 Register中的Two Retimers Presence Detected bit设置为1b，否则将其设置为0b
- 如果Device支持大于2.5 GT/s的速率，则它必须记录下它在link的任意lane上接收到的data rate identifier。这将覆盖以前接收到的值。在recovery阶段追踪speed change的变量，即changed_speed_recovery，将被重置为0b
- 如果Flit_Mode_Enabled为0b。如果该device发送TS2 OS，且其中的Link Upconfigure / L0p Capability设置为1b，而且在接受到的8个连续的TS2 OS中，Link Upconfigure / L0p Capability同样为1b，则将upconfigure_capable设置为1b，否则将其重置为0b
- 如果Flit_Mode_Enabled为1b，且LinkUp为0b。如果该device发送TS2 OS，且其中的Link Upconfigure / L0p Capability设置为1b，而且在接受到的8个连续的TS2 OS中，Link Upconfigure / L0p Capability同样为1b
- L0p_capable设置为1b
- Device Status 3 Register中的Remote L0p Supported bit设置为1b
- 所有的剩下的不是Configured Link一部分的lanes，与LTSSM不再关联，而且必须执行以下操作
- 如果支持该特性，则可选的，与一个新的crosslink LTSSM相关联
- 所有没有与新的crosslink LTSSM相关联的lanes，必须切换到Electrical Idle。且接收端必须满足。
- 对于升/降lane的情况，如果LTSSM advertise Link width upconfigure capability，则那些在L0 state中形成了link，且LinkUp为1b lane，他们虽然不再是当前配置的link的一部分，但是不允许将其与LTSSM解耦。建议这些lanes的receiver保持开启状态。但是如果关闭了的话，则如果upconfigure_capable为1b，则当LTSSM进入Recovery.Rcvrcfg，直到LTSSM进入Configuration.Complete阶段，它们必须开启，以允许可能的升lane操作。在最初的link training to L0过程中，并没有成为LTSSM一部分的lanes，在升lane过程中，也不允许成为link的一部分
- 当LTSSM切换回Detect阶段后，这些lanes必须立即与LTSSM重新关联
- 在切换到Electrical Idle之前，无需发送EIOS，且向EIOS的切换也无需发生于Symbol / OS的边界处
- 在2ms的timeout之后
- 如果当前速率为2.5 GT/s或者5.0 GT/s，则切换到Detect
- 如果当前速率为8.0 GT/s或者更高，且idle_to_rlock_transitioned小于FFh，则切换到Configuration.Idle
- 将changed_speed_recovery重置为0b
- 那些不是当前Configured Link的lanes，不再与LTSSM相关联
- 如果至少有一条lane接收到了8个连续的TS2 OS，且拥有符合的Link / L按哦number，则upconfigure_capable允许，但是不一定要更新。如果更新，则如果发送和接收的Link Upconfigure为1b，则将upconfigure_capable设置为1b，否则设置为0b
- 否则，切换到Detect
## Configuration.Idle
- 当使用8b/10b编码时，在Non Flit mode下，transmitter在所有configured lanes上发送Idle data symbols，而在Flit mode下，发送IDLE flits
- 如果LinkUp=0b，且link的所有部分都支持64.0 GT/s的速率（在进入Configuration.Idle之前的8个连续的TS2 OS或者8个连续且相似的Modified TS2 OS中体现），则：
- 如果在进入Configuration.Idle之前接收到的8个连续的TS2 OS或者8个相似的Modified TS2 OS中，以及自己在所有configured lanes上发送出去的Modified TS2 OS中，No Equalization Needed bit为1b。或者在接收到的8个连续的TS2 OS中，以及自己发送出去的TS2 OS中，Link Behavior Control field中的值为10b，即No Equalization Needed
- equalization_done_8GT_data_rate, equalization_done_16GT_data_rate, equalization_done_32GT_data_rate, and equalization_done_64GT_data_rate都设置为1b
- 64.0 GT/s Status Register中的No Equalization Needed Received bit为1b
- 如果在进入Configuration.Idle之前接收到的8个连续的TS2 OS或者8个相似的Modified TS2 OS中，以及自己在所有configured lanes上发送出去的Modified TS2 OS中，Bypass to Highest NRZ Rate bit为1b。或者在接收到的8个连续的TS2 OS中，以及自己发送出去的TS2 OS中，Link Behavior Control field中的值为01b/10b，即为No Equalization Needed / Equalization Bypass to Highest NRZ Rate
- equalization_done_8GT_data_rate, equalization_done_16GT_data_rate为都设置为1b
- 如果进入是因为接收到了8个连续且相似的Modified TS2 OS，且LinkUp=0b
- 如果接收到的8个Modified TS2 OS中，Modified TS Usage field设置为010b，即Alternate Protocols，并且发送出去的TS2 OS中也是如此，并且在所有的configured lanes上接收和发送的TS2 OS中，Modified TS Information1 / Alternate Protocol Vendor ID都是一样的
- 32.0 GT/s Status Register中的Modified TS Received bit设置为1b。具体的协商细节，将由接收到的8个Modified TS2 OS中的Received Modified TS Data 1 Register和Received Modified TS Data 2 Register来反映出来
- 如果LinkUp=0b，且link的所有部分都支持32.0 GT/s的速率（在进入Configuration.Idle之前的8个连续的TS2 OS或者8个连续且相似的Modified TS2 OS中体现），则：
- 如果在进入Configuration.Idle之前接收到的8个连续的TS2 OS或者8个相似的Modified TS2 OS中，以及自己在所有configured lanes上发送出去的Modified TS2 OS中，No Equalization Needed bit为1b。或者在接收到的8个连续的TS2 OS中，以及自己发送出去的TS2 OS中，Link Behavior Control field中的值为10b，即No Equalization Needed
- equalization_done_8GT_data_rate, equalization_done_16GT_data_rate, equalization_done_32GT_data_rate设置为1b
- 32.0 GT/s Status Register中的No Equalization Needed Received bit为1b
- 否则，如果在进入Configuration.Idle之前接收到的8个连续的TS2 OS或者8个相似的Modified TS2 OS中，以及自己在所有configured lanes上发送出去的Modified TS2 OS中，Bypass to Highest NRZ Rate bit为1b。或者在接收到的8个连续的TS2 OS中，以及自己发送出去的TS2 OS中，Link Behavior Control field中的值为01b/10b，即为No Equalization Needed / Equalization Bypass to Highest NRZ Rate
- equalization_done_8GT_data_rate, equalization_done_16GT_data_rate为都设置为1b
- 如果进入是因为接收到了8个连续且相似的Modified TS2 OS，且LinkUp=0b
- 如果接收到的8个Modified TS2 OS中，Modified TS Usage field设置为010b，即Alternate Protocols，并且发送出去的TS2 OS中也是如此，并且在所有的configured lanes上接收和发送的TS2 OS中，Modified TS Information1 / Alternate Protocol Vendor ID都是一样的
- 32.0 GT/s Status Register中的Modified TS Received bit设置为1b。具体的协商细节，将由接收到的8个Modified TS2 OS中的Received Modified TS Data 1 Register和Received Modified TS Data 2 Register来反映出来
- 在Non-Flit Mode中，Receivers等待Idle data，而在Flit Mode中等待IDLE Flits
- 当在Non Flit Mode下使用128b/130b编码时
- 如果data rate为8.0 GT/s，则transmitter在所有configured lanes上发送一个SDS OS，来开启Data Stream，而后在所有configured lanes上发送idle data symbol。lane 0上发送的第一个data symbol则是data stream的第一个data symbol
- 如果data rate为16.0 GT/s或者更高，则transmitter在所有configured lanes上，紧跟着SDS OS后发送一个control skp os，来开启Data Stream，而后在所有configured lanes上发送idle data symbol。lane 0上发送的第一个data symbol则是data stream的第一个data symbol
- 当在Flit Mode下使用128b/130b编码，或者使用1b/1b编码时
- transmitter在所有configured lanes上，发送一个SDS OS，后面紧跟着一个control skp os，来开启一个带有IDLE Flits的date stream
- 对于Receivers，它们在Non-Flit Mode下等待idle data symbol，而在Flit Mode下等待IDLE Flits
- 在此时，LinkUp为1b
- 当在Non Flit Mode下使用8b/10b编码时，如果在所有configured lanes上接收到了8个连续的Idle data，而且在接受到一个idle data symbol之后，发送出去了16个idle data symbol，则进入L0
- 如果自从上次从recovery或者Configuration进入L0时，软件已经在Link Control Register中的Retrain Link bit中写入了1b，则Downstream Port必须将Link Status Register中的Link Bandwidth Management Status bit设置为1b
- 在切换到L0时，必须将use_modified_TS1_TS2_Ordered_Set重置为0b
- 当在Flit Mode下使用128b/130b编码时，如果在所有configured lanes上，如果在所有configured lanes上接收到了8个连续的Idle data，而且在接受到一个idle data symbol之后，发送出去了16个idle data symbol，并且该substate不是由于Configuration.Complete的timeout进入的，则进入L0
- Idle data Symbols必须在Data Blocks中被接收到
- 在Data Stream处理开始之前，必须完成Lane-to-Lane de-skew
- 如果自从上次从recovery或者Configuration进入L0时，软件已经在Link Control Register中的Retrain Link bit中写入了1b，则Downstream Port必须将Link Status Register中的Link Bandwidth Management Status bit设置为1b
- 在切换到L0时，必须将idle_to_rlock_transitioned bit重置为00h
- 在Flit Mode下，如果接收到2个连续的IDLE Flits，而且在接收到一个IDLE Flit之后，发送出去了固定个数的IDLE Flits，且该substate不是由于Configuration.Complete的timeout进入的，则进入L0。对于固定个数的IDLE Flits，对于8b/10b或者128b/130b编码，为4个，而对于1b/1b编码，为8个
- 在Data Stream处理开始之前，必须完成Lane-to-Lane de-skew
- 如果自从上次从recovery或者Configuration进入L0时，软件已经在Link Control Register中的Retrain Link bit中写入了1b，则Downstream Port必须将Link Status Register中的Link Bandwidth Management Status bit设置为1b
- 在切换到L0时，必须将idle_to_rlock_transitioned bit重置为00h
- 否则，在最少2ms的timeout之后：
- 如果idle_to_rlock_transitioned小于FFh，则进入Recovery.RcvrLock
- 在切换到Recovery.RcvrLock的过程中
- 如果速率为8.0 GT/s或者更高，则将idle_to_rlock_transitioned的值加1
- 如果速率为2.5 GT/s或者5.0 GT/s，则将idle_to_rlock_transitioned的值设置为FFh，且将use_modified_TS1_TS2_Ordered_Set重置为0b
- 否则，切换到Detect
# Recovery
如果连接处于正常状态，那么Link会被直接训练到L0，而不会进入Recovery状态。下面描述几种进入Recovery状态的情况：
​
- 当离开L1时，没有fast training option（比如sending FTS OS）
​
- 当离开L0s时，receiver没有在规定时间内，使用FTS来获得Lock
​
- 当离开L0时，以下任意情况满足：
- 当最初的链路训练完成时，link可以工作于更高的速率
- 发起了一个Link speed change或者Link width change。原因是功耗管理，亦或当前的速率/宽度工作不稳定
- 软件将Link Control Register中的Retrain Link bit置为1，从而发起Link Retrain来解决传输错误
- DL层的错误，例如Replay Num Roll-over event，导致PL层自动开始链路重训练
- Receiver在任意Configured Lanes上见到了TS1s或者TS2s，这意味着link另一侧的device已经进入Recovery了
- Receiver在所有Configured Lanes上见到了Electrical Idle，但是在这之前并没有接收到Electrical Idle Ordered Set
Recovery的状态机如图所示：
注意，对于64.0 GT/s的速率，Pre-cursor指的是使用的2个pre-cursors
## Recovery.RcvrLock
如果Link工作于8.0 GT/s或者更高的速率，则必须要在lane获得Block Alignment之后，才能判断TS0 / TS1 / TS2 OS的接收。如果是从L1 / Recovery.Speed / L0s进入该substate的，则在退出Electrical Idle之后就必须要获得Block Alignment。而如果是从L0进入该substate的，则在最后一笔Data Stream的结尾之后，必须获得Block Alignment。
- 如果link工作于8.0 GT/s或者更高的速率
- 如果start_equalization_w_preset设置为1b
- 对于一个Upstream Port，当它开始在一个需要执行链路均衡的速率传输数据时，则它必须使用它在Recovery.RcvrCfg substate中接收到的8个连续的TS2 OS（如果是8.0 GT/s则为EQ TS2，如果是32.0 GT/s且启用了Equalization Bypass to Highest NRZ Rate则为EQ TS2，如果是16.0 GT/s或者32.0 GT/s或者64.0 GT/s则为128b/130b EQ TS2）中的transmitter preset values来配置它的transmitter settings，并且要确保符合对应的电气要求。对于那些接收到了一个reserved或者unsupported transmitter preset value的lanes，当它开始在一个需要执行链路均衡的速率传输数据时，则必须使用特定方法来选择一个transmitter preset value（其实这就对应了在Recovery.RcvrCfg substate中接收到的transmitter preset value不合法的情况）。
- 对于一个Downstream Port，当它开始在一个需要执行链路均衡的速率传输数据，则使用以下方法规定的自己的Transmitter Preset settings
1. 如果需要进行链路均衡的速率为16.0 GT/s，32.0 GT/s或者64.0 GT/s，并且在最近的一次通过Recovery.RcvrCfg过程中，接收到了8个连续的EQ TS2 OS（这对应于 equalization bypass to 32.0 GT/s的情况）或128b/130b EQ TS2 OS，且这些TS2 OS中带有supported Transmitter Preset values，则必须使用这些Transmitter Preset values
2. 否则，如果寄存器里存好的preset value有效的话，使用寄存器中预存的值，如图所示。使用的是Lane Equalization Control Register Entry中的Downstream Port Transmitter Preset field
1. 否则，使用特定机制来确定Transmitter Preset values
注意，无论采用哪种方式来选择transmitter preset values，Downstream选择的Transmitter Preset values必须符合PCIe定义的电气要求
- 而后，下一个state为Recovery.Equalization，即开始链路均衡的流程
- 如果start_equalization_w_prese为0b
- Transmitter必须使用最后一次链路协商到的coefficient settings
- 如果该substate是从Recovery.Equalization进入的，则在传输的TS1 OS中，Downstream Port必须将Pre-cursor，Cursor，以及Post-crusor的Coefficient fields设置为现在的transmitter settings。并且，如果Recovery.Equalizaton中的phase 2的最后一笔接收的request是一个preset request，则Downstream Port必须将Transmitter Preset bits设置为那笔request的accepted preset。
- 建议在该substate内，在传输的TS1 OS中，所有的port将Pre-cursor，Cursor，以及Post-crusor的Coefficient fields设置为现在的transmitter settings。并且将Transmitter Preset bits设置为最近一次Transmitter settings设置为的值
- 一个Upstream Port，如果接收到了8个连续的TS0或者TS1 OS，且符合以下特征，则进入Recovery.Equalization state
- 如果接收到的是8个连续的TS1 OS，Link / Lane number和自己的每条lane上发送出去的TS1 OS中的一致
- 如果接收到的是8个连续的TS1 OS，speed_change bit为0b
- 如果接收到的是8个连续的TS1 OS，EC bits不为00b
- 如果接收到的是8个连续的TS0 OS，且上一次进入Recovery.RcvrLock是从Recovery.Speed substate进入的
- 对于一个Downstream Port而言，如果Recovery.RcvrLock不是从Configuration.Idle或者Recovery.Idle进入的，且Link Control 3 Register中的Perform Equalization bit为1b或者使用特定的机制来决定需要执行链路均衡，则next state为Recovery.Equalization
- Downstream Port必须确保在Recovery.RcvrLock substate中，在切换到Recovery.Equalization且开始发送TS0/TS1 OS之前，发送的EC=00b的TS1 OS不超过2个
- Transmitter在所有configured lanes上发送TS1 OS，其中的link / lane number与离开Configuration时设置的一致。如果directed_speed_change为1，则必须将speed_change bit设置为1。如果任意一条configured lane上，接收到8个连续的TS1 OS，且speed_change为1，则将directed_speed_change设置为1。仅仅那些大于2.5 GT/s，且能稳定工作的速率能够被advertise。在Non Flit Mode下，TS1 OS中的N_FTS的值反映了在当前工作速率下需要的FTS数量。当进入该substate时，允许device改变它advertise的支持速率
对于一个Downstream Port，当支持Equalization Bypass to Highest NRZ时，如果它想要在速率从2.5 GT/s或者5.0 GT/s切换到8.0 GT/s或者32.0 GT/s时进行重均衡，则必须：
- 发送EQ TS1 OS，且speed_change bit设置为1b，且advertise如下的data rates
- 如果是为了8.0 GT/s执行重均衡，则advertise 8.0 GT/s Data Rate Identifier
- 如果是为了32.0 GT/s执行重均衡，则advertise 32.0 GT/s Data Rate Identifier
- 如果链路重均衡是由硬件发起的，则硬件必须确保在发起重均衡之前，工作速率必须为2.5 GT/s或者5.0 GT/s
- 如果链路重均衡是由软件发起的，则软件必须确保在发起重均衡之前，工作速率必须为2.5 GT/s或者5.0 GT/s
对于一个Downstream Port，如果它想要在速率从8.0 GT/s切换到16.0 GT/s，或者从16.0 GT/s切换到32.0 GT/s，或者从32.0 GT/s切换到64.0 GT/s时，执行链路重均衡，则必须：
- 发送TS1 OS，且Equalization Redo bit设置为1b，speed_change bit设置为1b，而且advertise需要执行均衡的data rates（16.0 GT/s，32.0 GT/s，64.0 GT/s）
- 如果链路重均衡是由硬件/软件发起的，则硬件/软件必须确保在发起重均衡之前，工作速率必须符合下列条件：
- 如果要在16.0 GT/s进行重均衡，则当前速率必须为8.0 GT/s
- 如果要在32.0 GT/s进行重均衡，则当前速率必须为16.0 GT/s
- 如果要在64.0 GT/s进行重均衡，则当前速率必须为32.0 GT/s
对于一个Upstream Port
## Recovery.Equalization
## Recovery.Speed
- Transmitter进入Electrical Idle，并且一直停留在Electrical Idle，直到Receiver Lanes也进入Electrical Idle状态。而且，对于成功的速率协商（successful_speed_negotiation=1b），transmitter将额外在Electrical Idle停留至少800ns，而对于失败的速率协商（successful_speed_negotiation=0b），将额外停留至少6us。但是，不允许Transmitter在Electrical Idle停留超过1ms。只有在Receiver Lanes进入Electrical Idle之后，才允许工作速率切换到新的速率。如果协商的速率为5.0 GT/s，且工作于full swing模式，则需要考虑发送方的去加重设置。如果select_deemphasis为0b，则选择-6 dB的去加重，而如果select_deemphasis为1b，则选择-3.5dB的去加重。注意，如果link已经工作于双方port所支持的最高速率了，仍然会执行Recovery.Speed，但是工作速率不会改变
在进入Electrical Idle之前，必须发送一个EIESQ出去
DC common voltage无需处于spec规定的范围内
对于一个lane，如果它接收到了一个EIOS，或者它推断/监测到Electrical Idle，我们就说这个lane处于Electrical Idle
- 如果在一次成功的速率协商之后，进入Recovery.Speed。则如果在规定的时间间隔中，没有收到TS1 OS或者TS2 OS，我们就认为lane进入Electrical Idle。这种情况对应于link是optional的，且link两边都能成功接收到TS OS的情况。因此，在规定的时间间隔中，没有收到TS1或者TS2 OS则可以被理解为electrical idle
- 如果在一次失败的速率协商之后，进入Recovery.Speed，当没有在规定的时间间隔中，至少接收到一个Exit from Electrical Idle，则认为lane处于Electrical Idle。这种情况对应于至少link有一端不能接收另一端发来的TS OS，因此在一个长间隔内，没有收到Exit from Electrical Idle便被视作进入Electrical idle
- 如果符合以下情景，即Transmitter Lanes无需再处于Electrical Idle状态了，则进入Recovery.RcvrLock
- 如果Recovery.Speed是从Recovery.RcvrCfg进入的，而且有一个成功的速率协商，例如successful_speed_nogotiation=1b。则在所有的configured lanes上，将速率切换到最大的共同支持速率。而且将changed_speed_recovery设置为1b
- 否则，如果自从LTSSM从L0或者L1进入Recovery之后，已经是第二次进入Recovery.Speed了（即changed_speed_recovery已经为1b了），则将速率切换到从L0/L1进入Recovery时的速率，而后将changed_speed_recovery重置为0b
- 否则，如果在进入Recovery.Speed之前，使用的最新编码是1b/1b，则将速率切换到32.0 GT/s，而后将changed_speed_recovery重置为0b
- 否则，将速率切换到2.5 GT/s，而后将changed_speed_recovery重置为0b
注意，这对应于在L0下的速率大于2.5 GT/s，但是link一端在那个速率下不能工作，并且在从L0或者L1第一次进入Recovery.RcvrLock时，在Recovery.Rcvrlock state timeout了
- 如果在进入Recovery.Speed时的速率小于64.0 GT/s，则经过48ms的timeout之后，进入Detect
- 在normal conditions，这种state转换不可能发生
- 在96ms的timeout之后，进入Detect。如果在进入Recovery.Speed时的速率大于等于64.0 GT/s，则在48ms的timeout之后，进入Detect
- 强烈建议使用96ms的timeout。因为在64.0 GT/s下，48ms的timeout会在不正确的情况下将LTSSM导向Detect。这是因为64.0 GT/s Recovery.Equalization Phase 2 timeout为64ms，大于Phase 1 timeout 12ms + Recovery.Speed timeout 48ms
- 在Recovery.Speed中，directed_speed_change将会被重置为0b。而new data rate需要反应在Link Status Register中的Current Link Speed field
- 在Link bandwidth切换中，如果successful_speed_negotiation设置为1b，并且在Recovery.RcvrCfg中接收到的8个连续的TS2 OS中的Autonomous Change bit设置为1b，或者speed change是Downstream Port自动发起的，原因是工作不可靠，则将Link Status Register中的Link Autonomous Bandwidth Status bit设置为1b
- 否则，在Link bandwidth change中，将Link Status Register中的Link Bandwidth Management Status bit设置为1b
## Recovery.RcvrCfg
在Non Flit Mode下，Transmitter在所有configured lanes上发送TS2 OS，其中的Link / Lane number和离开Configuration时设置的一致。
在Flit Mode下，Transmitter在所有configured lanes上发送TS2 OS。其中Link number作如下规定：
- 如果LTSSM是为了Link Width Change而从Recovery.RcvrCfg切换到Configuration，且这些lane将要被从Link中去除，则将Link number fields设置为PAD，否则还是将Link number设置为当前的Link number
- 如果speed_change为1b，或者LTSSM的切换路径是从Recovery.Equalization -> Recovery.RcvrLock -> Recovery.RcvrCfg，则LTSSM不允许发起Link Width Change
- 如果directed_speed_change已经设置为1b了，则speed_change bit必须设置为1b
在发送的TS2 OS中，对于Non Flit Mode，N_FTS的值必须反映出当前工作的速率
在Flit Mode下，如果port想要在离开该substate后将LTSSM切换到Hot Reset / Disabled / Loopback，则发送的TS2 OS中对应的Training Control bit必须被set
对于Downstream Port，如果下列条件都满足，则它必须在每个configured lanes上发送EQ TS2 OS，其中Transmitter Preset和Receiver Preset Hint fields必须设置为对应的Lane Equalization Control Register Entry中的Upstream 8.0 GT/s Port Transmitter Preset和Upstream 8.0 GT/s Port Receiver Preset Hint fields中的值
- Downstream Port在Recovery.RcvrLock中，advertise 8.0 GT/s support，而且在离开Detect State后，Upstream port已经在Configuration.Complete或者Recovery.RcvrCfg substates中advertise 8.0 GT/s data rate support，并且在进入Recovery.RcvrCfg之前，在任意configured lanes上接收到8个连续的TS1 OS或者TS2 OS，其中speed_change bit为1b
- equalization_done_8GT_data_rate为0b，或者Link Control 3 Register中的Perform Equalization bit为1b，或者由特定的机制决定需要进行链路均衡
- 当前的工作速率为2.5 GT/s或者5.0 GT/s
对于Downstream Port，如果下列条件都满足，则它必须在每个configured lanes上发送EQ TS2 OS，其中Transmitter Preset必须设置为对应的32.0 GT/s Lane Equalization Control Register Entry中的32.0 GT/s Upstream Port Transmitter Preset fields中的值，而Receiver Preset Hint field设置为000h
- Downstream Port在Recovery.RcvrLock中，advertise 32.0 GT/s support，而且在离开Detect State后，Upstream port已经在Configuration.Complete或者Recovery.RcvrCfg substates中advertise 32.0 GT/s data rate support，并且在进入Recovery.RcvrCfg之前，在任意configured lanes上接收到8个连续的TS1 OS或者TS2 OS，其中speed_change bit为1b
- equalization_done_32GT_data_rate为0b，或者Link Control 3 Register中的Perform Equalization bit为1b，或者由特定的机制决定需要进行链路均衡
- Equalization_done_8GT_data_rate和Equalization_done_16GT_data_rate均为1b
- 在Configuration State中，link两端已经协商好了Equalization Bypass to Highest NRZ Rate
- 当前的工作速率为2.5 GT/s或者5.0 GT/s
对于Downstream Port，如果下列条件都满足，则它必须在每个configured lanes上发送128b/130b EQ TS2 OS，其中Transmitter Preset必须设置为对应的16.0 GT/s Lane Equalization Control Register Entry中的16.0 GT/s Upstream Port Transmitter Preset fields中的值
- Downstream Port在Recovery.RcvrLock中，advertise 16.0 GT/s support，而且在离开Detect State后，Upstream port已经在Configuration.Complete或者Recovery.RcvrCfg substates中advertise 16.0 GT/s data rate support，并且在进入Recovery.RcvrCfg之前，在任意configured lanes上接收到8个连续的TS1 OS或者TS2 OS，其中speed_change bit为1b
- equalization_done_16GT_data_rate为0b，或者Link Control 3 Register中的Perform Equalization bit为1b，或者由特定的机制决定需要进行链路均衡
- 当前工作速率为8.0 GT/s
对于Downstream Port，如果下列条件都满足，则它必须在每个configured lanes上发送128b/130b EQ TS2 OS，其中Transmitter Preset必须设置为对应的32.0 GT/s Lane Equalization Control Register Entry中的32.0 GT/s Upstream Port Transmitter Preset fields中的值
- Downstream Port在Recovery.RcvrLock中，advertise 32.0 GT/s support，而且在离开Detect State后，Upstream port已经在Configuration.Complete或者Recovery.RcvrCfg substates中advertise 32.0 GT/s data rate support，并且在进入Recovery.RcvrCfg之前，在任意configured lanes上接收到8个连续的TS1 OS或者TS2 OS，其中speed_change bit为1b
- equalization_done_32GT_data_rate为0b，或者Link Control 3 Register中的Perform Equalization bit为1b，或者由特定的机制决定需要进行链路均衡
- 当前工作速率为16.0 GT/s
对于Downstream Port，如果下列条件都满足，则它必须在每个configured lanes上发送128b/130b EQ TS2 OS，其中Transmitter Preset必须设置为对应的64.0 GT/s Lane Equalization Control Register Entry中的64.0 GT/s Upstream Port Transmitter Preset fields中的值
- Downstream Port在Recovery.RcvrLock中，advertise 64.0 GT/s support，而且在离开Detect State后，Upstream port已经在Configuration.Complete或者Recovery.RcvrCfg substates中advertise 64.0 GT/s data rate support，并且在进入Recovery.RcvrCfg之前，在任意configured lanes上接收到8个连续的TS1 OS或者TS2 OS，其中speed_change bit为1b
- equalization_done_64GT_data_rate为0b，或者Link Control 3 Register中的Perform Equalization bit为1b，或者由特定的机制决定需要进行链路均衡
- 当前工作速率为32.0 GT/s
对于Upstream Port，如果下列条件都满足，则允许它发送128b/130b EQ TS2 OS，其中16.0 GT/s Transmitter Preset bits设置为由具体实现决定的值
- Upstream Port在Recovery.RcvrLock中，advertise 16.0 GT/s support，而且在离开Detect State后，Downstream port已经在Configuration.Complete或者Recovery.RcvrCfg或者Recovery.RcvrLock（更推荐） substates中advertise 16.0 GT/s data rate support，并且在进入Recovery.RcvrCfg之前，在任意configured lanes上接收到8个连续的TS1 OS或者TS2 OS，其中speed_change bit为1b
- equalization_done_16GT_data_rate为0b，或者由特定机制决定要进行链路均衡
- 当前工作速率为8.0 GT/s
对于想要bypass equalization to the highest data of 32.0 GT/s或者更高速率的Upstream Ports，如果以下条件满足，则它们必须发送8b/10b EQ TS2 OS，其中32.0 GT/s Transmitter Preset bits设置为由具体实现决定的值
- 在Configuration state中，已经成功协商了equalization bypass to the highest NRZ rate
- Upstream Port需要precoding，或者Upstream Port想要提供Downstream Port的starting 32.0 GT/s Transmitter Preset
- Upstream Port在Recovery.RcvrLock中，advertise 32.0 GT/s support，而且在离开Detect State后，Downstream port已经在Configuration.Complete或者Recovery.RcvrCfg substates中advertise 32.0 GT/s data rate support，并且在进入Recovery.RcvrCfg之前，在任意configured lanes上接收到8个连续的TS1 OS或者TS2 OS，其中speed_change bit为1b
- equalization_done_32GT_data_rate为0b，或者由特定机制决定要进行链路均衡
- 当前工作速率为2.5 GT/s或者5.0 GT/s
对于Upstream Port，如果以下条件满足，则允许发送128/130b EQ TS2 OS，其中32.0 GT/s Transmitter Preset bits设置为由具体实现决定的值
- Upstream Port在Recovery.RcvrLock中，advertise 32.0 GT/s support，而且在离开Detect State后，Downstream port已经在Configuration.Complete或者Recovery.RcvrCfg substates或者Recovery.RcvrLock中advertise 32.0 GT/s data rate support，并且在进入Recovery.RcvrCfg之前，在任意configured lanes上接收到8个连续的TS1 OS或者TS2 OS，其中speed_change bit为1b
- equalization_done_32GT_data_rate为0b，或者由特定机制决定要进行链路均衡
- 当前工作速率为16.0 GT/s
对于Upstream Port，如果以下条件满足，则允许发送128/130b EQ TS2 OS，其中64.0 GT/s Transmitter Preset bits设置为由具体实现决定的值
- Upstream Port在Recovery.RcvrLock中，advertise 64.0 GT/s support，而且在离开Detect State后，Downstream port已经在Configuration.Complete或者Recovery.RcvrCfg substates或者Recovery.RcvrLock中advertise 64.0 GT/s data rate support，并且在进入Recovery.RcvrCfg之前，在任意configured lanes上接收到8个连续的TS1 OS或者TS2 OS，其中speed_change bit为1b
- equalization_done_64GT_data_rate为0b，或者由特定机制决定要进行链路均衡
- 当前工作速率为32.0 GT/s
当使用128b/130b编码，或者1b/1b编码时，Upstream Ports和Downstream Ports使用TS2 OS中的Request Equalization / Equalization Request Data Rate / Quiesce Guarantee bits来互相交流equalization requests。当没有request equalization时，Request Equalization / Equalization Request Data Rate / Quiesce Guarantee bits都必须被设置为0b
当进入Recovery.RcvrCfg时，必须将start_equalization_w_preset重置为0b
- 当进入该substate时，Downstream Port必须将select_deemphasis设置为Link Control 2 Register中的Selectable De-emphasis field中的值，或者采用特定的方法来规定select_deemphasis的值（包括使用Upstream Port发送出来的8个连续的TS1 OS中的值）。一个advertise 5.0 GT/s data rate support的Downstream Port，必须将TS2 OS中的De-emphasis的值设置为select_deemphasis的值。如果一个Upstream Port因为autonomous reasons，想要改变Link bandwidth，则它必须将TS2 OS中的Autonomous Change bit设置为1b
- 对于支持Link width upconfigure的device，如果directed_speed_change设置为0b，则建议在该substate / Recovery.Idle / Configuration.Linkwidth.Start中，在inactive lane中开启Electrical Idle detection电路。这样做，从而在Configuration.Linkwidth.Start中发起的Link Upconfigure过程中，没有发起升lane的side不会错过发起升lane的side发送过来的第一个EIEOSQ
- 如果以下条件全部满足，则切换到Recovery.Speed
- 以下条件有一个满足
- 在任意configured lanes上，接收到8个连续的TS2 OS，这些TS2 OS具有相同的data rate identifier， 在symbol 6具有相同的值，而且speed_change bit都为1b。无论使用8b/10b还是128b/130b编码，这些TS 2 OS都是标准的TS2 OS
- 在所有configured lanes上，接收到8个连续的EQ TS2 OS或者128b/130b EQ TS2 OS，这些TS2 OS具有相同的data rate identifier， 在symbol 6具有相同的值，而且speed_change bit都为1b
- 在任意configured lanes上，接收到8个连续的EQ TS2 OS或者128b/130b EQ TS2 OS，这些TS2 OS具有相同的data rate identifier， 在symbol 6具有相同的值，而且speed_change bit都为1b，并且在任意configured lane上接收到8个EQ OS之后已经过了1ms
- 当使用1b/1b编码时，在任意configured lane上，接收到8个连续且相同的TS2 OS，其中speed_change bit设置为1b
- 当前速率大于2.5 GT/s，或者在发送出去的TS2 OS和接收到的8个连续的TS2 OS中，存在大于2.5 GT/s data rate identifier
- 对于8b/10b编码，在同一个configured lane上，当接收到一个TS 2 OS，且其中speed_change为1b之后，已经发送出去了至少32个TS 2 OS，其中speed_change为1b。注意32个TS2 OS不能被EIEOS打断。对于128b/130b编码，在同一个configured lane上，当接收到一个TS 2 OS，且其中speed_change为1b之后，已经发送出去了至少128个TS 2 OS，其中speed_change为1b.
在接收到的8个连续的TS2 OS中，且speed_change bit为1，其中advertise的data rates就是other port所支持的data rates。接收到的8个连续的TS2 OS中的Autonomous Change bit需要被Downstream Port寄存下来，因为可能会在Recovery.Speed substate中写入到Link Status Register中。而对于Upstream Port，它必须寄存接收到的8个连续的TS2 OS中的Selectable De-emphasis到自己的select_deemphasis变量中。在Recovery.Speed中需要切换到的新速率是Link两边的port所共同支持的最大速率。
对于一个Upstream Port，如果当前速率为2.5 GT/s或者5.0 GT/s，而且接收到的8个连续的TS2 OS为EQ TS2 OS，且advertise 8.0 GT/s as the highest data rate supported，则它必须将start_equalization_w_preset设置为1b，并且使用对应lane接收到的TS2 OS中的值来更新Lane Equalization Control Register Entry中的Upstream 8.0 GT/s Transmitter Preset和Upstream 8.0 GT/s Receiver Preset Hint fields
对于一个Upstream Port，如果当前速率为2.5 GT/s或者5.0 GT/s，而且接收到的8个连续的TS2 OS为EQ TS2 OS，且advertise 32.0 GT/s as the highest data rate supported，且在Configuration阶段，link两端的port已经成功协商了equalization bypass to the highest NRZ rate，则它必须将start_equalization_w_preset设置为1b，并且使用对应lane接收到的TS2 OS中的值来更新32.0 GT/s Lane Equalization Control Register Entry中的Upstream 32.0 GT/s Transmitter Preset
对于一个Upstream Port，如果当前速率为8.0 GT/s，link两端都advertise support for 16.0 GT/s，而且接收到的8个连续的TS2 OS是128b/130b EQ TS2 OS，则必须将start_equalization_w_preset设置为1b，并且使用对应lane接收到的TS2 OS中的值来更新16.0 GT/s Lane Equalization Control Register Entry中的Upstream 16.0 GT/s Transmitter Preset
对于一个Upstream Port，如果当前速率为16.0 GT/s，link两端都advertise support for 32.0 GT/s，而且接收到的8个连续的TS2 OS是128b/130b EQ TS2 OS，则必须将start_equalization_w_preset设置为1b，并且使用对应lane接收到的TS2 OS中的值来更新32.0 GT/s Lane Equalization Control Register Entry中的Upstream 32.0 GT/s Transmitter Preset
对于一个Upstream Port，如果当前速率为32.0 GT/s，link两端都advertise support for 64.0 GT/s，而且接收到的8个连续的TS2 OS是128b/130b EQ TS2 OS，则必须将start_equalization_w_preset设置为1b，并且使用对应lane接收到的TS2 OS中的值来更新64.0 GT/s Lane Equalization Control Register Entry中的Upstream 64.0 GT/s Transmitter Preset
任意configured lanes，如果没有接收到符合上述要求的EQ TS2或者128b/130b EQ TS2 OS，当首次工作于8.0 / 16.0 / 32.0 / 64.0 GT/s时，在执行链路均衡之前，需要使用与具体实现相关的preset值。对于一个Downstream Port，如果以下条件任意为真，则必须将start_equalization_w_preset设置为1b
- equalization_done_8GT_data_rate为0b
- 两边都advertise support for 16.0 GT/s，且equalization_done_16GT_data_rate为0b
- 两边都advertise support for 32.0 GT/s，且equalization_done_32GT_data_rate为0b
- 两边都advertise support for 64.0 GT/s，且equalization_done_64GT_data_rate为0b
- Link Control 3 register中的Perform Equalization bit为1b
- 采用与具体实现有关的机制，来决定必须执行链路均衡
对于一个Downstream Port，如果接收到的8个连续的TS2 OS是128b/130b EQ TS2 OS，且port两边都advertise 16.0 GT/s，32.0 GT/s，64.0 GT/s，则它必须寄存下来TS2 OS中的16.0 GT/s，32.0 GT/s，64.0 GT/s Transmitter Preset settings。successful_speed_negotiation设置为1b。注意，如果link已经工作于link两端所支持的最大速率了，则仍然会进入Recovery.Speed，但不会执行速率切换。如果使用128b/130b或者1b/1b编码，而且在接收到的8个连续的TS2 OS中，将Request Equalization bit设置为1b，则port必须将其判断为equalization request。
- 如果以下2个条件均满足，则进入Recovery.Idle
- 在所有configured lanes都接收到了8个连续的TS2 OS，且Link / Lane number都与对应的lane一致，每个lane上接收到的TS2 OS的data rate identifier一致，而且以下条件有一个满足
- 在接收到的8个连续的TS2 OS中，speed_change bit为0b
- 当前速率为2.5 GT/s，而且在接收和发送的TS2 OS中，data rate identifier中没有5.0 GT/s或者更高速率
- 在接收到1个TS2 OS之后，发送出去了16个TS2 OS，且没有被任何EIEOS打断。在进入Recovery.Idle时，将changed_speed_recovery / directed_speed_change重置为0b
- 在Flit Mode下，如果接收到的8个连续的TS2 OS中，在任意configured lane上，Hot Reset bit或者Disable Link bit或者Loopback bit为1，则对应的directed bit必须为1
- 上述要求确保了link两端的components立即从Recovery.Idle退出，进入对应的Hot Reset / Disable / Loopback state，而不需要follower发送一个data stream，而后等待下一个SKP OS边界
- 如果N_FTS的值已经改变了，则对于以后的L0s states，必须使用新值
- 当使用8b/10b编码时，必须在离开Recovery.RcvrCfg之前，完成Lane-to-Lane de-skew
- 在state transtion中，device必须记录下8个TS2 OS中的data rate identifier，这会覆盖以前的值
- 当使用128b/130b编码或者1b/1b编码时，如果在接收到的8个连续的TS2 OS中，Request Equalization为1，则device必须将其记录。
- 在Non-Flit Mode下，如果在任意configured lane上接收到8个连续的TS1 OS，且其中的Link / Lane number和这些lane上正在发送的TS OS中的不一致，并且在接收到1个TS1 OS之后，已经发送出去了16个TS2 OS，则无论使用8b/10b编码还是128b/130b编码，当以下2个条件满足时，进入Configuration
- 在接收到的TS1 OS中，speed_change为0b
- 当前速率为2.5 GT/s，而且在接收和发送的TS2 OS中，data rate identifier中没有5.0 GT/s或者更高速率
- 如果LTSSM切换到Configuration，则changed_speed_recovery和directed_speed_change会被重置为0b
- 如果N_FTS的值已经改变了，则对于以后的L0s states，必须使用新值
- 在Flit Mode下，如果以下所有条件都满足，则切换到Configuration
- 在该substate中，在任意configured lanes上接收到一个TS1或者TS2 OS之后，过了1ms的timeout
- 下列条件有一个满足：
- 在任意configured lanes上接收到一个TS2 OS之后，又接收到了8个连续的TS1 OS
- 在任意configured lanes上没有接收到TS2 OS，而是接收到了8个连续的TS1 OS，且他们的Link或者Lane number为PAD
- 在任意configured lanes上接收到TS2 OS，且Link number为PAD，或者在任意configured lanes上正在发送TS2 OS，且Link / Lane number为PAD
- 在接收到下列任意一组OS后，至少发送了16个TS2 OS出去
- 一个TS1 OS，且Link或者Lane number设置为PAD
- 一个TS2 OS
- 在接收到的TS1 OS或者TS2 OS中，speed_change bit为0b
如果LTSSM切换到Configuration，则将changed_speed_recovery和directed_speed_change都重置为0b
在Flit Mode下，没有Upconfig support，因此从Recovery -> Configuration的路线是为了由于error而导致的降lane。在这种情况下，想要降lane的port应该在那些不想再使用的lane上发送TS OS，且其中的link number为PAD
- 在以下条件满足时，进入Recovery.Speed
- 自从LTSSM从L0或者L1切换到Recovery之后，操作速率已经成功切换到了两边协商的速率，例如changed_speed_recovery为1b
- 在任意configured lane上，探测到了一个EIOS，或者推断出了Electrical Idle
- 自从进入该substate之后，没有configured lane接收到一个TS2 OS
- 在离开Recovery.Speed之后，新的操作的速率将回到从L0或者L1进入Recovery时的速率
- 在以下条件满足时，进入Recovery.Speed
- 自从LTSSM从L0或者L1切换到Recovery之后，操作速率没有成功切换到了两边协商的速率，例如changed_speed_recovery为0b
- 目前工作的速率大于2.5 GT/s
- 在任意configured lane上，探测到了一个EIOS，或者推断出了Electrical Idle
- 自从进入该substate之后，没有configured lane接收到一个TS2 OS
- 在离开Recovery.Speed之后，如果使用8b/10b编码，则工作速率为2.5 GT/s。如果使用1b/1b编码，则工作速率为32.0 GT/s
- 注意，这种transition暗示着link另一端的port无法在工作速率获得symbol lock或者block alignment。因此，link两端将要回到2.5 GT/s，而且除非退出Recovery state，link两端的port都不会再请求改变速率。注意，尽管这里包含了一个speed change，但是changed_speed_recovery会是0b
- 在48 ms超时之后
- 如果当前速率为2.5 GT/s或者5.0 GT/s，则切换到Detect
- 如果当前速率为8.0 GT/s或者更高，且idle_to_rlock_transitioned小于FFh，则切换到Recovery.Idle
- 在进入Recovery.Idle时，将changed_speed_recovery和directed_speed_change重置为0b
- 否则，切换到Detect
## Recovery.Idle
- 如果受到引导，则进入Disabled
- 对于Downstream Port，引导意为更高层将Link发送到的TS1 / TS2中的Disable Link bit设置为1
- 对于Upstream Port，在Flit Mode下，引导意味着在切换到该substate时接收到的8个连续的TS2 OS中的Disable Link bit为1
- 如果受到引导，则进入Hot Reset
- 对于Downstream Port，引导意为更高层将Link发送到的TS1 / TS2中的Hot Reset bit设置为1
- 对于Upstream Port，在Flit Mode下，引导意味着在切换到该substate时接收到的8个连续的TS2 OS中的Hot Reset bit为1
- 如果收到引导，则进入Configuration
- 引导意味着port收到更高层的指示，来reconfigure link，例如切换link宽度
- 如果收到引导，且transmitter能够成为Loopback Lead（这是通过与具体实现决定的），则进入Loopback
- 引导意味着port收到更高层的指示，来将Link发送的TS1 / TS2的Loopback bit设置为1b，且该port成为loopback lead
- 在Non Flit Mode下，如果任意configured lane接收到2个连续的TS1 OS，其中Disable Link bit为1，则进入Disabled
- 在Non Flit Mode下，如果任意configured lane接收到2个连续的TS1 OS，其中Hot Reset bit为1，则进入Hot Reset
- 如果任意configured lane上，接收到2个连续的TS1 OS，且Lane number为PAD，则进入Configuration
- 如果Port是为了切换Link Configuration而进入Configuration state，则它保证在所有的lane上发送lane number为PAD
- 建议在Non Flit Mode下，LTSSM采用这种切换流程来升/降lane，从而减少时间消耗。在Flit Mode下，应该使用Recovery.RcvrCfg to Configuration切换流程来升/降lane
- 如果以下条件有一个为真，则切换到Loopback
- Port工作于Non Flit Mode，而任意configured lane在2个连续的TS1 OS中，将Loopback bit设置为1
- Port工作于Flit Mode，在任意lane上，在进入该state时接收到的8个连续的TS2 OS中，Loopback bit为1
- 注意，接收到Ordered Set，且Loopback bit为1的device，是Loopback Follower
- 当使用8b/10b编码时，在Non Flit Mode下，Transmitter在所有的Configured Lanes上发送Idle data Symbols。在Flit Mode下，Transmitter在所有的Configured Lanes上发送IDLE Flits
- 当在Non Flit Mode下，使用128b/130b编码时：
- 如果data rate为8.0 GT/s，则Transmitter在所有configured lanes上发送一个SDS OS，来开启Data stream的传输。而后在所有configured lanes上发送Idle data Symbols。Lane 0上发送的第一个Idle data symbol是整个Data Stream的第一个data symbol
- 如果data rate为16.0 GT/s及以上，则Transmitter在所有configured lanes上首先发送一个Control SKP OS，而后立即发送一个SDS OS，来开启Data stream的传输。而后在所有configured lanes上发送Idle data Symbols。Lane 0上发送的第一个Idle data symbol是整个Data Stream的第一个data symbol
- 如果要切换到其他state，而非L0，则不允许发送Idle data symbols
当切换到Configuration / Loopback / Hot Reset / Disabled时，如果Data stream是active的（例如已经发送出去了一个SDS OS），则必须首先发送一个EDS。有可能没有发起升lane的port在接收到TS1 OS时，它已经发送了SDS OS，而且已经在发送Data stream了。在那种情况下，在Configuration state发送TS1 OS之前，它必须在lanes上发送EDS
- 当在Flit Mode下，使用128b/130b编码或者1b/1b编码时：
- transmitter在所有configured lanes上首先发送一个Control SKP OS，而后立即发送一个SDS OS，而后发送IDLE Flits来开启Data Stream的传输。
- 如果被引导到其他states，则不允许发送IDLE Flits
- 当在Non Flit Mode下使用128b/130b编码，如果Recovery.Idle不是由于timeout而从Recovery.RcvrCfg切换而来，而且在所有lanes上都接收到了8个连续的symbol times的Idle data，则进入L0。在接收到1个Idle Data Symbol之后，必须发送16个Idle Data Symbol出去。
- Idle data symbols必须在Data blocks中被接收
- 在开始Data stream的处理之前，必须完成Lane-to-lane de-skew
- 如果自从上次从Recovery / Configuration切换到L0之后，软件已经将Link Control Register中的Retrain Link bit设置为1，则Downstream Port必须将Link Status Register中的Link Bandwidth Management Status bit设置为1b
- 当切换到L0时，必须将idle_to_rlock_transitioned重置为00h
- 在Flit Mode下，如果Recovery.Idle不是由于timeout而从Recovery.RcvrCfg切换而来，并且接收到了2个连续的IDLE Flits，而且在接收到一个IDLE Flit之后，又发送出去了一定数量的IDLE Flits，则切换到L0。对于8b/10b或者128b/130b编码，IDLE Flits数量为4，而对于1b/1b编码，数量为8
- 在开始Data stream的处理之前，必须完成Lane-to-lane de-skew
- 如果自从上次从Recovery / Configuration切换到L0之后，软件已经将Link Control Register中的Retrain Link bit设置为1，则Downstream Port必须将Link Status Register中的Link Bandwidth Management Status bit设置为1b
- 当切换到L0时，必须将idle_to_rlock_transitioned重置为00h
- 否则，在2ms的timeout之后
- 如果idle_to_rlock_transitioned小于FFh，则切换到Recovery.RcvrLock
- 如果data rate为8.0 GT/s或者更高，当切换到Recovery.RcvrLock时，将idle_to_rlock_transitioned加1
- 如果data rate为5.0 GT/s或者2.5 GT/s，当切换到Recovery.RcvrLock时，将idle_to_rlock_transitioned重置为FFh
- 否则，切换到Detect
# L0
L0是常规的，完全工作状态下的Link state。在该logica state下，link两端的device交换TLPs和DLLPs。在链路训练完成之后，就进入L0状态。同时，PL层也会将LinkUp设置为1，以此通知更上层，link已经准备好运行了。L0同时包含了L0p state，在L0p下，有的lane能够处于Electrical Idle状态
- LinkUp=1b，且在收到STP或者SDP symbol时，将idle_to_rlock_transitioned清空为00h
- 对于Upstream Port，如果从Detect state之后，在Configuration.Complete或者Recovery.RcvrCfg substates中从没有记录下Downstream Port advertise大于2.5 GT/s的速率，则directed_speed_change不允许被设置为1b（其实就是不允许速率切换，因为仅仅支持2.5 GT/s）
- 对于Downstream Port，如果从Detect state之后，在Configuration.Complete或者Recovery.RcvrCfg substates中从没有记录下Upstream Port advertise大于2.5 GT/s的速率，则directed_speed_change不允许被设置为1b（其实就是不允许速率切换，因为仅仅支持2.5 GT/s）。
- 对于Downstream Port，如果它记录下超过2.5 GT/s的速率了，如果Link Control Register中的Retrain Link bit为1b且Link Control 2 Register中的Target Link Speed不等于当前运行的速率，则必须将directed_speed_change设置为1b（其实就是开始升速率的过程）
- 支持2.5 GT/s以上速率的ports，即使没有处于DL_Active state，如果它被link另一端的port通过TS OS请求速率切换，则它必须参与速率切换。其实就是一侧的component的DL层还没有初始化完成，但是link另一侧的component请求速率切换，则必须响应速率切换
- 更高层将directed_speed_change设置为1b，发起change speed。且满足以下任意条件，则切换到recovery：
- link两端都支持大于2.5 GT/s的速率，且link处于DL_Active state
- Link两端都支持大于8.0 GT/s的速率，且想要在那个速率下进行链路均衡，即目的是通过链路均衡后，将changed_speed_recovery bit重置为0b
- Downstream port选择了一个alternate protocol，但是目前的operation data rate不支持协商好的alternate protocol，即目标是切换速率，从而允许协商alternate protocol
- 由于想要改变link width，而进入recovery
- 当link另一端的port在Configuration阶段没有advertise自己支持upconfigure link width，或者Link目前已经工作在协商的最大宽度，则upper layer不允许在link这端的port上发起升lane
- 通常情况下，如果upconfigure_capable重置为0b时，除非是因为可靠性原因，upper layer不会降lane，因为当upconfigure_capable为0b时，link降lane后就不能升lane到以前的宽度了
- 如果Link Control Register中的Hardware Autonomous Width Disable bit为1，则除非是因为可靠性原因，upper layer不会降lane
- 如果在任意configured lane上接收到TS1 / TS2 OS，或者在128b/130b或者1b/1b编码下，在任意configured lane上接收到EIEOS，则进入recovery
- 如果Upper layer要求，则LTSSM可以切换到Recovery。如果在任意lane上都没有收到EIOS，但是在所有lanes上都detect / infer到Electrical Idle，则Port可能会切换到Recovery，也可能仍然停留在L0。注意，对于仍然停留在L0的情况，由于没有收到EIOS就发生了Electrical Idle，可能会发生错误，并且Port被upper layer引导进入recovery
- 在以下情况下，可能会在所有lanes上infer出Electrical Idle：
- 在任意一个128 us的窗口内，没有传输Flow Control Update
- 在任意一个128 us的窗口内，没有传输SKP OS
- 在任意一个128 us的窗口内，没有传输Flow Control Update DLLP / Optimized_Update_FC
- if directed，指的是Port被更高层指示，从而进入Recovery。方法包括set Link Control Register中的Retrain Link bit
- 在切换到Recovery时，Transmitter可能会完成任何in progress的TLP或者DLLP
- 对于Transmitter，仅仅当其实现了L0s，且更高层让它切换到L0s时，才会进入L0s状态
- if directed，指的是Port被更高层指示，从而进入L0s
- 在这种情况下，Rx和Tx有可能会进入不同的LTSSM states
- 对于Receiver，仅仅当receiver实现了L0s，且接收到了一个EIOS，且没有被更高层设置为L1或者L2时，才会进入L0s
- 在这种情况下，Rx和Tx有可能会进入不同的LTSSM states
- 如果在任意Lane上接收到了EIOS，且receiver没有实现L0s，且Port没有被higher layers引导进入L1或者L2，且EIOS并不是L0p降宽进程的一部分，则进入Recovery
- 以下是进入L1的步骤：
1. link两边协商完成，当EIOS的接收和发送条件满足时，进入L1
2. link一侧的device1受到更高层指示，在所有lanes上发送EIOS，而后进入Electrical Idle
3. 另一侧的device2接收到EIOS，而后在所有lanes上发送EIOSQ，而后立即进入L1
4. device1等待任意lane上接收到EIOS，而后进入L1
- 以下是进入L2的步骤：
1. link两边协商完成，当EIOS的接收和发送条件满足时，进入L2
2. link一侧的device1受到更高层指示，在所有lanes上发送EIOS，而后进入Electrical Idle
3. 另一侧的device2接收到EIOS，而后在所有lanes上发送EIOSQ，而后立即进入L1
4. device1等待任意lane上接收到EIOS，而后进入L1
# L0s
L0s是一种低功耗状态，且拥有最低的返回L0的延迟。Device使用硬件控制进入或者退出L0s。注意，Link两端的device都允许独立进入或者退出L0s
L0s的发送端和接收端状态机如图所示：
## Receiver L0s
如果port advertise它支持L0s，则receiver必须实现L0s，这是通过receiver的Link Capabilities Register中的ASPM Supported field来指示的。同时，即使Port没有advertise支持L0s，也允许receiver去实现L0s
## Rx_L0s.Entry
- 在T timeout之后，进入Rx_L0s.Idle
- 这是transmitter必须处于electrical idle的最低时间
## Rx_L0s.Idle
- 如果receiver在configured link的任意lane上detect到了exit from Electrical Idle，则进入Rx_L0s.FTS，即进入唤醒流程
- 如果当前速率为8.0 GT/s或者更高，并且port的receiver不满足2.5 GT/s下的Z，则在100 ms的timeout之后，进入Rx_L0s.FTS.当速率为8.0 GT/s或者更高时，所有的port都允许实现这个timeout，并且切换到Rx_L0s.FTS
## Rx_L0s.FTS
- 在link的所有configured lanes上，如果使用8b/10b编码并且接收到了一个SKP OS，或者使用128b/130b编码并且接收到了一个SDS OS，则进入L0
- 在8b/10b编码下，receiver在接收到SKP OS之后，必须立即能够接收valid data
- 在128b/130b编码下，receiver在接收到SDS OS之后，必须立即能够接收valid data
- 在离开Rx_L0s.FTS之前，就必须完成Lane-to-Lane de-skew
- 否则，在N_FTS timeout之后，进入recovery
- 当使用8b/10b编码时， N_FTS timeout必须小于，而不长于2倍的该时间。当Extended Synch为1时，N_FTS的范围必须在。当在spec规定的范围内考虑N_FTS的选取时，具体实现必须考虑最坏情况下的Lane to Lane deskew，design margins，以及4到8个连续的EIE symbols（在2.5 GT/s之外的速率）
- 当使用128b/130b编码时，对于8.0 GT/s和16.0 GT/s，N_FTS必须小于，不长于2倍的该时间。对于32.0 GT/s以及以上的速率，N_FTS必须小于，并且不长于2倍的该时间。
- 对于这种情况，transmitter必须同样允许进入recovery，但是允许先完成任何正在进行的TLP或者DLLP
- 建议在切换到Recovery时，增加N_FTS，从而防止以后再次从Rx_L0s.FTS切换到recovery
## Transmitter L0s
如果port advertise它支持L0s，则transmitter必须实现L0s，这是通过transmitter的Link Capabilities Register中的ASPM Supported field来指示的。同时，即使Port没有advertise支持L0s，也允许transmitter去实现L0s
## Tx_L0s.Entry
- Transmitter发送EIOSQ，并且进入Electrical Idle
- DC common vlotage必须处于spec规定的范围内
- 在T timeout之后，进入Tx_L0s.Idle
## Tx_L0s.Idle
- 如果directed，则进入Tx_L0s.FTS
- 由于Rx_L0s.FTS中的timeout，从而增加N_FTS：transmitter通过在Tx_L0s.Idle中发送N_FTS个Fast Training Sequence，来让receiver获得bit lock和symbol lock或者block alignment。如果没有收到足够的Fast Training Sequence，则receiver将会在Rx_L0s.FTS substate超时，可且可能会在recovery state中增加自己advertise的N_FTS数量
## Tx_L0s.FTS
- Transmitter必须在所有configured lanes上发送N_FTS个Fast Training Sequences
- 在5.0 GT/s的速率下，在发送N_FTS个FTS之前（如果Extended Synch bit为1的话，则发送4096个FTS），必须发送4-8个EIE symbols。在128b/130b编码下在发送N_FTS个FTS之前，必须发送1个EIEOSQ（如果Extended Synch bit为1的话，则发送4096个FTS）。在2.5 GT/s的速率下，在发送N_FTS个FTS之前（如果Extended Synch bit为1的话，则发送4096个FTS），则可能会发送最多一个full FTS
- 在所有的FTS发送完成之前，不允许发送SKP OS。FTS的总数由N_FTS参数的协商而定
- 如果Extended Synch bit为1，则transmitter必须发送4096个FTS，按照spec的要求来插入SKP OS
- 当使用8b/10b编码时，transmitter必须在所有configured lanes上发送一个单独的SKP OS
- 当使用128b/130b编码时，transmitter必须在所有configured lanes上发送一个EIEOSQ，后面跟着一个SDS OS。注意，在Lane 0上传输的，紧跟着SDS OS的第一个symbol，是data stream的第一个symbol
- 在完成了上述的transmission之后，进入L0
- 在16.0 GT/s以及更高速率下退出L0s时，无需插入SKP OS：与其他LTSSM state不同，当退出Tx_L0s.FTS时，在发送SDS之前，无需发送control SKP OS。这会导致前一个datastream的最后一部分相关的Data Parity information被忽略。无需发送Control SKP OS，减少了设计复杂度，并且改善了exit latency性能。
# L1
与L0s状态相比，L1具有更激进的功耗管理措施，因此拥有更大的exit latency。L1是ASPM的一个选项，这意味着它和L0s一样，硬件能直接控制device进入和退出L1而无需软件介入。但是，与L0s不同的是，软件也能直接让一个Upstream Port发起LTSSM转换，到L1，这是通过将device power state写到更低的level（D1 / D2 / D3）来实现的。同时，与L0s不同的是L1状态同时影响Link两端的device。
既然进入Electrical Idle可能预示着link partner想要进入L0s，L1或者L2，到底进入哪个state是由link两端的device先前的协商结果决定的。
L1 state的状态机如图所示：
## L1.Entry
- 所有configured transmitters都处于Electrical Idle状态
- DC common voltage必须处于SPEC规定的电压范围内
- 在时间之后，跳转到L1.Idle状态
- 这段等待的时间，保证了transmitter已经成功过渡到Electrical Idle condition
## L1.Idle
- Transmitter仍然处于Electrical Idle状态
- DC common voltage必须处于SPEC规定的电压范围内，除非是启用了L1 PM substates（在L1.2 substate中，可以无需维持DC common voltage）
- 当满足L1 PM Substate的条件时，进入L1的substate
- 当进入或者退出L1.Idle时，L1 PM substate必须是L1.0
- 在以下情况下，离开L1.Idle，进入Recovery
- 在Configured Link的任意一条lane上，检测到了exit from Electrical Idle
- 在2.5 GT/s以外的其他速率中，在该substate至少停留40ns，而后被upper layer引导进入recovery
- 注意，在2.5 GT/s以外的其他速率中，至少需要在该substate停留40ns，这是为了考虑到logic levels的delay，从而允许激活electrical idle detection电路。这是为了link进入L1，而后立即退出L1的情况而实现的
- Port允许采用和L0中一样的机制，先将directed_speed_change设置为1b，而后开启切速度的进程。在这一过程中，changed_speed_recovery必须设置为0b
- Port允许通过recovery进入L0，而后将directed_speed_change设置为1b，将LTSSM切换到Recovery，开启切速率进程
- port允许像L0中一样，由上层指引进入Recovery，开启切宽度的进程
- 如果当前速率为8.0 GT/s或者更高速率，并且port的receiver不满足2.5 GT/s下的z，则在100ms的timeout之后，进入recovery。对于所有的port，允许但不强制，在8.0 GT/s以及更高的速率下实现这个timeout机制并且切换到recovery
- 这个timeout机制不受L1 PM substates机制的影响
- L1下的100ms timeout：满足2.5 GT/s的Z的ports，他们在L1.Idle下无需实现100ms timeout，因此也无需因此切换到recovery。这种port需要阻止实现这种机制，因为这种机制会减少power saving的功效。
# L2
L2是比L1更激进的功耗节省状态，同时它也有更大的exit latency。当Device处于D3 Cold Power state，且合适的握手流程已经完成时，Power Management 软件引导Upstream Port进入L2。
当系统得知一切已经准备就绪时，它将切断main power。当切断power后，如果第二power source，即存在，则Link进入L2，如果不存在，则进入L3
L2的作用是使用来自的small power，当事件发生，需要link恢复main power时，能够通知system。有两种通知机制：
- 使用一个称为WAKE# pin的side band信号
- 使用WAKE#信号，不一定必须实现L2
- 使用一个称为Beacon的in band信号
- 如果使用可选的Beacon信号，则必须实现L2
- 一般只有2.5 GT/s的速率下使用Beacon
L2的状态机如图所示：
## L2.Idle
- 所有的receivers都必须在1ms内，满足2.5 GT/s下的
- 所有的configured transmitters，都必须停留在Electrical Idle状态下，时间
- DC common voltage无需处于SPEC规定范围内
- 对于receiver，至少等待，而后开始检测Electrical Idle Exit
- 对于Downstream Ports
- 对于所有的Downstream Ports，如果至少在Lane0上接收到了Beacon，或者由更高层导向，则切换到Detect
- 在进入Detect之前必须首先恢复main power
- 对于Upstream Ports
- 如果在任何一条先前配置好的lane上，探测到lElectrical Idle Exit，则进入Detect
- 所谓预先决定好的lane，即predetermined lanes，必须包含但不限于那先有可能协商为link lane 0的lane。对于multi-lane links，lane的数量必须大于等于2
- 如果更高层指引Upstream Port发送一个Beacon，则其进入L2.TransmitWake
- 注意，Beacon仅仅只能在Upstream Ports上发送至RC
## L2.TransmitWake
该state仅仅适用于Upstream Ports
- 在该state下，Upstream Ports至少在Lane0上发送Bacon
- 如果在任意Upstream port的receiver上，检测到Electrical Idle Exit，则进入Detect
- 在检测到Electrical Idle Exit之前，要保证main power已经恢复
# Disabled
Disabled Link的概念是，该Link处于Electrical Idle状态，且无需保持DC common voltage。软件将Link Control Register中的Link Disable bit设置为1，而后device发送出TS1s，其中的Link Disable bit为1
- 建议一进入Disabled state，就将LinkUp设置为0，而无需等待发送EIOSQ / 接收EIOS
- 所有的lanes发送16～32个TS1 OS，其中Disable Link bit为1，而后进入Electrical Idle
- 在进入Electrical Idle之前，必须发送EIOSQ
- DC common voltage无需处于spec规定范围内
- 如果已经发送出去了EIOSQ，且任何lane接收到了一个EIOS，则：
- 将LinkUp设置为0b
- 在此时，lane被视为disabled
- 对于Upstream Ports：所有的receivers都必须在1ms内，满足2.5 GT/s下的
- 对于Downstream Ports：当至少在一个lane上检测到Electrical Idle Exit，则进入Detect
- 对Downstream Ports：由更上层将其导向Detect（如软件将Link Disable bit重置为0）
- 对于Upstream Ports：如果在2ms timeout之后，没有接收到EIOS，则进入Detect
# Loopback
Loopback阶段的状态机如图所示：
Loopback是一个test and debug feature，在正常的操作流程中不会使用。充当Loopback Master的device，通过发送TS1 OS，且其中的Loopback bit为1，来将Link Partner置于Loopback slave模式。
在Loopback下，Loopback master向Loopback发送valid Symbols，而后Loopback slave将其反射回来。注意在这种情况下，Loopback slave继续进行时钟补偿，因此master必须继续以合适的间隔插入SOS，而Loopback slave负责在SOS中插入或者删除SKP OS。
当Loopback master发送一个EIOS（而后Loopback Master会进入Electrical Idle），而receiver检测到Electrical Idle之后，退出Loopback state。
## Loopback.Entry
- 在此状态下，LinkUp=0b
- 在此状态下，receiver忽略接收到的TS1和TS2 OS中的Link number和Lane number
- 对于Loopback Leader的要求：
- 如果Loopback Lead在进入Loopback.Entry之前，且LinkUp=0b时，被上层引导，需要在一个active lane上（该lane被称为lane under test）进行64.0 GT/s的链路均衡流程，则必须将内部的寄存器值perform_equalization_for_loopback_64GT设置为1b
- 如果Loopback.Entry是从Recovery.Equalization进入的，并且当前工作速率为32.0 GT/s，并且perform_equalization_for_loopback_64GT为1b且equalization_done_64GT_data_rate为0b，则需要决定Lead所支持的最大工作速率，以及Loopback Follower所支持的最大工作速率（由它在Recovery.Equalization的Phase 1中，在Lane under test上，发送的TS1 OS中advertise的data rates决定）。如果共同支持的最大速率为64.0 GT/s，则Loopback Lead发送16个连续的TS1 OS，其中Loopback bit为1，而后发送一个EIESQ，而后切换到Electrical Idle并维持1ms。在Electrical Idle阶段，将速率切换到64.0 GT/s
在速率切换到64.0 GT/s之前，发送的16个TS1 OS有如下要求：
- Enhanced Link Behavior Control bits必须为01b。对于所有的均衡流程，都必须针对同一条Lane under test进行
- 如果Loopback需要在not under test的lane上发送Modified Compliance Pattern，则Transmit Modified Compliance Pattern in the Loopback bit必须为1b
- 如果Loopback.Entry是从Configuration.Linkwidth.Start进入的，当LinkUp=0b时，Loopback Lead想要工作于64.0 GT/s，且Loopback也知道Loopback Follower也支持64.0 GT/s，则hignest common data rate为64.0 GT/s。否则，通过Loopback Lead所支持的最大速率，以及在切换到Loopback.Entry时，在任意一个active lane上接收到的2个连续的TS1或者TS2 OS所advertise的data rates，来决定link两端所共同支持的最大速率。如果当前Loopback Leader工作的速率不是所支持的共同最大速率，则：
- 发送16个连续的TS1 OS，其中Loopback bit为1，而后发送一个EIESQ，而后切换到Electrical Idle并维持1ms。在Electrical Idle阶段，如果perform_equalization_for_loopback_64GT bit设置为1，且共同支持的最大速率为64.0 GT/s，则将速率切换到32.0 GT/s。否则将速率切换到共同支持的最大速率。（因为如果想要切换到64.0 GT/s的速率，且需要在64.0 GT/s下进行链路均衡，则必须首先切换到32.0 GT/s）
- 如果LinkUp为0b，如果PCIe Capabilities Register中的Flit Mode Supported bit为1b，且支持64.0 GT/s，且从最后一次Polling.Configuration切换来时，接收到的TS2 OS中的Flit Mode Supported bit为1b（这其实也就意味着Link对端的device也支持Flit Mode），则发送的16个TS1 OS中，Data Rate Identifier中的Supported Link Speeds field必须使用Flit Mode data rate encoding。
- 在进入Loopback.Entry之前，Loopback Lead可能会被特定的机制引导，来在一个active lane，即lane under test上进行32.0 GT/s的链路均衡。如果共同支持的最大速率为32.0 GT/s，且equalization_done_32GT_data_rate为0b，并且将要执行链路均衡，则在切换到最大的共同速率之前发送的16个连续的TS1 OS有如下要求：
- Enhanced Link Behavior Control bits必须为01b。
- 如果Loopback需要在没有进行测试的lane上发送Modified Compliance Pattern，则Transmit Modified Compliance Pattern in the Loopback bit必须为1b
- 如果共同支持的最大速率为5.0 GT/s，通过设置Loopback Leader所发送的TS1 OS中的Selectable De-emphasis bit来控制follower的transmitter de-emphasis，其实就是控制de-emphasis值为-3.5dB还是-6dB
- 对于5.0 GT/s及以上速率，lead允许使用特定的方式来选择自己的transmitter settings，而与自己发送给Follower的settings无关。
- Note：如果Loopback是LinkUp=1b之后进入的，则有可能link两端，一个Port是从Recovery进入Loopback而另一个Port是从Configuration进入Loopback。从Configuration进入Loopback的port可能会请求速率切换，而另一个Port不会。如果出现这种情况，则结果未知。在测试过程中应该避免这种情况的出现。
- 对于Lead发送的TS1 OS的另外要求：
- 如果是从Recovery.Equalization进入的Loopback.Entry，则发送的TS1 OS中的EC field必须为00b
- Lead允许将Loopback.Entry中传输的TS1 OS中的Compliance Receive bit设置为1，包括那些在data rate change之前发送出去的TS1 OS。如果asset Compliance Receive bit，则在Loopback.Entry state中不允许deassert。这个使用模型在Port两边速率切换后，难以获得bit lock / symbol lock / block alignment，以方便测试时很有用
- 如果data rate切换到了32.0 GT/s或者64.0 GT/s，且在任意lane上发送了16个TS1 OS，且Enhanced Link Behavior Control bits为01b，则切换到Recovery.Equalization。
- 注意将perform_equalization_for_loopback设置为1b（其实就是去做链路均衡了）
- 如果发送的TS1 OS中，Compliance Receive bit为1b，则2ms后进入Loopback.Active
- 如果Loopback.Entry是从Recovery.Equalization进入的，并且lane under tes上t接收到了2个连续的TS1 OS，且Loopback bit为1，则进入Loopback.Active
- 如果发送出去的TS1 OS中，Compliance Receive bit为0b，并且有一组与具体实现有关的lanes上接收到了2个连续的TS1 OS，且Loopback bit为1，则进入Loopback.Active。
如果速率切换已经完成，但是没有在32.0 GT/s或者64.0 GT/s上进行链路均衡，则lead必须考虑到Follower可能处于Electrical Idle的时长，并且向Follower发送足够多的TS1 OS，从而使得Follower能够在进入Loopback.Active之前能够获得Symbol lock / Block lock。当工作速率为8.0 GT/s及以上时，这些TS1 OS可以被用来配置Loopback Follower的transmit settings（通过将EC field设置为10b/11b（根据follow的port方向是DSP还是USP而定），并且给出一组preset或者valid coefficients）
Lane Numbering With 128b/130b encoding in loopback：如果当前工作速率使用128b/130b编码，而且link两端并没有协商Lane numbers，则有可能Loopback Lead和Follower并不能分别正确解码他们收到的信息，因为他们的lane使用不同的LSFR种子（其实就是由于lane reversal等原因，link两端的lane number对不上，例如USP的Lane 16连接到DSP的Lane0）.这种情况，可以通过在direct Lead to Loopback之前，允许Lead和Follower协商lane number来解决，亦或指示Lead 在Loopback.Entry中，assert Compliance Receive bit，或者使用其他的手段，来保证Link两端的LSFR seed一致。
- 在以上条件均不满足时，在一个由具体实现决定的，低于100ms的timeout之后，进入Loopback.Exit
- 对于Loopback Follower的要求：
- 如果Loopback.Entry是从Configuration.Linkwidth.Start进入的，且LinkUp为0b。并且引导follower进入Loopback.Entry的TS1 OS中，advertise 64.0 GT/s support，Enhanced Link Behavior Control为01b，且follower想要工作于64.0 GT/s。则perform_equalization_for_loopback_64GT必须为1b
- 如果Loopback.Entry是从Recovery.Equalization进入的，且当前速率为32.0 GT/s，且perform_equalization_for_loopback_64GT为1b而equalization_done_64GT_data_rate为0b。而且对于Loopback Follower，它在Recovery.Equalization的phase 1中发送的TS1s中advertise 64.0 GT/s，则Loopback Follower：
- 发送一个EIESQ，而后进入Electrical Idle state，并维持2ms，在Electrical Idle状态中，将速率切换到64.0 GT/s
- 如果Loopback Follower是一个Downstream Port，则它在发送EIESQ之前，发送16个连续的TS1 OS。且这16个TS1 OS必须将Equalization Control bits设置为00b
- 切换到Recovery.Equalization
- 对于32.0 GT/s和64.0 GT/s的链路均衡过程， ‘Lane under test’, perform_equalization_for_loopback variable, transmit_modified_compliance_pattern_in_loopback variable and the Link and Lane numbers必须保持一致
- 其实本段就是描述了在64.0 GT/s下进行Loopback测试之前进行的一些step。就是先进行32.0 GT/s下的均衡并且切换到32.0 GT/s，而后进行64.0 GT/s下的均衡，切到64.0 GT/s
- 如果Loopback.Entry是从Configuration.Linkwidth.Start进入的，则检测Link两端所支持的最大共同速率，从Follower所支持的最大速率，以及Follower在进入Loopback.Entry时任何active lanes上收到的2个连续的TS1或者TS2中所描述的速率（其实这就是Loopback Lead所advertise的它自己支持的最大速率）来判断。如果当前速率不是所支持的共同最大速率，则：
- 发送一个EIESQ，而后切换到Electrical Idle，并维持2ms。在Electrical Idle阶段，如果perform_equalization_for_loopback_64GT bit设置为1，且共同支持的最大速率为64.0 GT/s，则将速率切换到32.0 GT/s。否则将速率切换到共同支持的最大速率。其实就是先进32.0 GT/s，而后完成64.0 GT/s的链路均衡和速率切换
- 如果工作于full swing mode，且支持的共同最大速率为5.0 GT/s，则按照接收到的，将follower引导到Loopback.Entry的TS1 OS中的Selectable De-emphasis bit来设置自己（即Loopback Follower）transmitter的de-emphasis，如果Selectable De-emphasis为1b，则选择-3.5dB的De-emphasis，否则选用-6dB的De-emphasis
- 如果支持的共同最大速率为8.0 GT/s或者更高，且Loopback Lead使用EQ TS1 OS来将Follower引导到Loopback.Entry，则Loopback Follower使用接收到的EQ TS1 OS中的Preset field来设置自己的transmitter参数。如果支持的共同最大速率为8.0 GT/s或者更高，但使用常规的TS1 OS来将Follower引导到Loopback.Entry，则follower允许使用它默认的transmitter设置
- 如果Loopback.Entry是从Configuration.Linkwidth.Start进入的，且共同支持的最大速率为32.0 GT/s或者更高，且引导Follower进入Loopback.Entry的TS1 OS中的Enhanced Link Behavior Control bits为01b，则进入Recovery.Equalization
- perform_equalization_for_loopback为1b，指示在Recovery.Equalization中要为Loopback做均衡
- 如果引导Follower进入Loopback.Entry的TS1 OS中的Transmit Modified Compliance Pattern in Loopback为1b，则transmit_modified_compliance_pattern_in_loopback为1b
- 当从Recovery.Equalization进入Loopback.Entry时，接收到两个连续的TS1 OS的lane，且在Configuration.Linkwidth.Start中TS1 OS中Enhanced Link Behavior Control bits为01b，就是执行Loopback和Recovery.Equalization的lane
- Loopback Follower必须使用特定的方式来确定好一个有效的Link number。将Loopback Follower引导进入Recovery.Equalization的test measurement equipment，必须使用与具体实现有关的方法，来确保equipment自己使用和Loopback Follower相同的lane number，从而使得link两端lane number分配一致，因而LSFR的种子一致而不失调。
- 如果引导Follower进入Loopback.Entry的TS1 OS中的Compliance Receive bit为1b，则切换到Loopback.Active
- Follower的transmitter不需要在任意边界处转为发送looped-back date回去的状态，而是允许直接结束任何自己正在发送的Ordered Set
- 否则，Follower发送TS1 OS，且其中的Link / Lane number都为PAD
- 如果Loopback.Entry是从Recovery.Equalization进入的
- 发送的TS1s的EC field必须为00b
- 当接收到两个连续的TS1 OS，且Loopback bit为1b，则切换到Loopback.Active
- 否则，如果以下任意条件满足，则切换到Loopback.Active
- 速率为2.5 GT/s或者5.0 GT/s，并且在所有active lanes上都获得了symbol lock
- 速率为8.0 GT/s或者更高时，且在所有的active lanes上都接收到了两个连续的TS1 OS。接收到的TS1 OS所描述的equalization settings，如果EC field的值符合follower的port方向，且requested setting是preset或者一组valid coefficients，则该settings必须被Loopback Follower评估，并且应用到transmitter上。可选的，follower可以接收所有的EC field values。 如果应用了该settings，则它们必须在被接收到的500ns内生效，并且不允许导致transmitter违反电气要求超过1ns。与Recovery.Equalization不同的是，新的settings不会反应在follower所发送的TS1 OS中。
- 当使用8b/10b编码时，Follower的transmitter必须在Symbol边界处转为发送looped-back data，但是允许截断任何正在发送的Ordered Set。使用128b/130b编码时，follower的transmitter不需要一定在边界处转为发送looped-back data，并且允许截断任何正在发送的Ordered Set
## Loopback.Active
- Loopback Lead必须发送有效的经过编码的数据。除非是想要退出Loopback，否则Loopback Lead不允许发送EIOS。当工作于128b/130b编码时，Loopback Leads必须遵守4.2.2.6
- 当工作于1b/1b编码时，当开启Data Stream时，Loopback Lead必须遵循4.2.3.1。但是，对于Flit中的CRC和FEC，允许使用任意值来填充。如果自从离开Detect State之后，已经进入过L0，即已经进行过link宽度协商，则使用该宽度，否则默认使用x1宽度
- 如果transmit_modified_compliance_pattern_in_loopback bit为1，则从Recovery.Equalization进入Loopback的Loopback Follower，必须在那些在Detect.Active阶段检测到receiver，但是没有under test的lanes上发送Modified Compliance Pattern，否则这些lanes必须切换到Electrical Idle状态。而那些under test的lanes，则必须遵守下面的规则。注意，state的切换仅仅基于lane under test
- Loopback Follower必须将接收到的encoded information发送回去，同时注意考虑到在Polling阶段协商好的电平反转，同时继续执行时钟补偿
- 当工作于1b/1b编码时，Loopback Follower必须追踪Data Stream何时开始，并且将Data Stream和OS区分开来
- 如果自从离开Detect State之后，已经进入过L0，即已经进行过link宽度协商，则使用该宽度，否则默认使用x1宽度
- 对于Upstream Ports，SRIS mode由它在Configuration state接收到的TS1 OS决定
- 对于Downstream Ports，SRIS mode由自己的Link Control Register的SRIS Clocking bit决定
- SKPs必须以lane为基础进行增减。而且所有lane上SKPs的增减无需同步进行
- 对于8b/10b编码，如果SKP OS的回传需要增加SKP symbols，则在COM后的SKP symbol临近插入
- 对于8b/10b编码，如果SKP OS的回传需要减少SKP symbols，则直接不回传这个SKP symbol即可
- 对于128b/130b编码，如果SKP OS的回传需要增加SKP symbols，则在SKP_END symbol之前插入4个SKP symbol
- 对于128b/130b编码，如果SKP OS的回传需要减少SKP symbols，则在SKP_END symbol之前去除4个SKP symbol
- 对于1b/1b编码，如果SKP OS的回传需要增加SKP symbols，则在SKP_END symbol之前插入8个SKP symbol
- 对于1b/1b编码，如果SKP OS的回传需要减少SKP symbols，则在SKP_END symbol之前去除8个SKP symbol
- 除了电平反转外，Loopback Follower不允许对接收到的数据进行任何改变，即使该数据为invalid encoding。
- 如果以下有一个为真，则Exit to Loopback.Exit
- 由更高层引导进入Loopback.Exit，或者接收到了4个连续的EIOS
- 可选的，如果当前速率为2.5 GT/s，且任意lane上接收到了一个EIOS，或者在任意lane上检测/推断出了Electrical Idle
- Note：Electrical Idle的推断方法：任意configured lanes之一处于electrical idle状态128us，且在整个128us的时间窗口内，没有检测到exit from Electrical Idle
- 对于Loopback Follower，在它接收到EIOS的1ms内，必须要能够在任意lane上检测Electrical Idle
- 在接收到EIOS和真正进入Electrical Idle的时间间隔内，Loopback Follower可能接收到未定义的bit流，这是transmitter loopback回来的
- 不适用于这种case，因为Loopback Follower可能在接收到到EIOS 1ms后还是检测不到electrical idle
- 对于Loopback.Lead，如果更高层引导进入，则Exit to Loopback.Exit
## Loopback.Exit
- 不管当前的Link Speed，Loopback lead发送如下OS，而后在2ms内进入Electrical Idle
1. 对于仅支持2.5 GT/s的ports，发送1个EIOS，也可以发送8个连续的EIOS
2. 对于支持2.5 GT/s以上速率的ports，发送8个连续的EIOS
- 在发送出最后一个EIOS之后，Loopback Lead必须在所有lane上，在内切换到valid Electrical Idle condition。
- Note：EIOS有助于标记Loopback Lead上发生的Loopback send和compare操作的logical end。对于Loopback Lead，它在EIOS后接收到的任何data都必须被抛弃，因为这些数据是未定义的。
- Loopback Follower必须在2ms之内，进入Electrical Idle
- 在进入Electrical Idle之前，Loopback Follower必须将它探测到Electrical Idle之前接收到的所有Symbols都Loopback回去。这确保了Loopback Lead能够看到EIOS，从而标记任意Loopback send和compare操作的logical end。
- 对于Loopback Lead和Loopback Follower，下一个state是Detect
# Hot Reset
- 对于由更上层引导进入Hot Reset的lanes：
- Configured Link中的所有的Lanes发送TS1，且其中Hot Reset bit为1，Link / Lane number和配置一致
- 如果在任意lane上，接收到了2个连续的TS1，且其中Hot Reset bit为1，Link / Lane number和配置一致，则
- LinkUp设置为0b
- 如果没有更上层要求PL层保持在Hot Reset state，则进入Detect
- 否则Configured Link中的所有的Lanes继续发送TS1，且其中Hot Reset bit为1，Link / Lane number和配置一致
- 否则，在2ms的timeout之后，进入Detect
- 对于由其他情况进入Hot Reset的lanes（如接收到了2个连续的TS1 OS，且Hot Reset bit为1）：
- LinkUp设置为0b
- Configured Link中的所有的Lanes发送TS1，且其中Hot Reset bit为1，Link / Lane number和配置一致
- 如果在任意lane上，接收到了2个连续的TS1，且其中Hot Reset bit为1，Link / Lane number和配置一致，则重置2ms的计时器
- 否则，在2ms的timeout之后，进入Detect
总的来说，一般对于Downstream Ports，他们的lane由更上层导向Hot Reset。对于Upstream Ports，它们因为接收到两个连续的TS1，且其中Hot Reset bit为1，Link / Lane number和配置一致，从而从Recovery.Idle state切换到Hot Reset