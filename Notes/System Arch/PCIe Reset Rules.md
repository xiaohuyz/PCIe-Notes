# PCIe Reset Rules

本部分描述了PCIe Reset机制。本部分包含了SPEC中描述的PCIe架构机制和reset机制，但是不包含PCIe Conventional Reset和具体组件/平台的Reset之间的关系

## Conventional Reset

conventional reset包含了除了function level reset之外的所有reset机制。spec中规定了2中conventional reset: fundamental reset和所有其他的非fundamental reset。

在所有的设备形式和系统软件配置中，在某种意义上，必须要有一个硬件机制来将所有的port states恢复到本spec所规定的初始状态，这种机制便被称为fundamental reset。这种机制可以通过系统向组件/转换卡提供的额外信号来实现，这种信号被称为PERST#，并且这种信号必须符合第四章所述的电气需求。当PERST#被提供给一个组件/转化卡时，该信号必须被该组件/转换卡用作fundamental reset。当PERST#没有被提供给组件/转换卡时，fundamental reset由组件/转换卡自动产生。当fundamental reset由组件/转换卡自动产生，并且他的供电是由平台提供的，当提供的能量大于该设备形式/系统的限额时，该组件/转换卡必须自动产生一个fundamental reset

- 存在3种类型的conventional reset: Cold，Warm和Hot

- 当对某个组件开始供电时，必须进行一次fundamental reset，这被称为Cold Reset
- 在某些情形下，fundamental reset机制有可能在没有移除-重新应用供电的情况下，被硬件触发。这被称为Warm reset
- 对于Warm reset和Cold reset的具体产生手段，spec未作定义
- 存在一种in-band方法，能在link中传输conventional reset。这被称为Hot Reset，具体细节在spec的第四章

存在一种in-band机制，软件能利用这种机制来强制一个link进入electrical idle，从而disable该link。Disable一个link，会导致Downstream components进入hot reset

- 从任意一种conventional reset中退出时，所有的port registers和state machines都必须被重置为他们的初始值，但是sticky registers不会被重置

- 注意，从device的视角来看，任何类型的conventional reset（cold，warm，hot or DL_Down）对于Transaction Layer和它以上的逻辑，都有和传统PCI中做RST#的assertion和de-assertion一样的效果

- 当退出fundamental reset时，PL层将会尝试重建link。当link两端的components都进入初始的link training state，他们将经过PL层的link初始化流程，而后经过VC0的flow control流程，从而使DL层和TL层准备就绪

- 当进行VC0的flow control初始化流程时，link两端的components将会互相传输TLP和DLLP

在退出conventional reset时，有些device在能够回复接收到的request之前，可能需要额外的恢复时间。尤其是对于configuration requests，components和devices必须工作于确定性的条件下，即必须满足以下条件

针对components和devices的要求

- 对于一个支持5.0 GT/s以上速率的link，它必须在fundamental reset结尾之后的100ms内进入LTSSM Detect state。而对于仅仅支持5.0GT/s以及以下速率的link，必须在20ms以内。对于所有的components而言，该时间越短越好。该要求也适用于retimer

- 在某些系统中，link两端的components可能在不同时刻退出fundamental reset。每个component都是从各自退出fundamental reset的时间开始计时的

- 当Link Training完成时，即进入DL_Active state时，一个component必须能够接收和处理TLPs和DLLPs
- 对于一个device，在退出fundamental reset之后，他要在1s之内，能够接受configuration requests，并且如果requests有效，能够返回一个成功的completion。这个时间的要求与link training多块完成无关。如果使用了Readiness Notification机制，则该时间可能长于或者短于1s

针对系统的要求

- 为了让components执行内部初始化，系统软件必须在device执行fundamental reset之后，等待一段时间，而后才允许发送configuration requests给这些device，除非该系统启用了Readiness Notifications。如果系统软件以及提前得知了具体device的初始化时间需求，则可以不等待那么长时间。
- 如果devcie/function提示自己支持immediate readiness，在允许系统软件立即向其发送configuration requests
- 如果DRS Message Received bit被set，在允许系统软件立即向downstream port的device/function发送configuration request
- 支持flit mode的devices需要实现DRS机制
- 因为在Flit Mode下，必须实现DRS机制，系统软件可以通过Downstream Port中，Downstream Component Presence和Flit Mode Status fields来决定Device是否支持DRS。对于系统软件无法决定devcie是否支持DRS：

- 对于不支持大于5.0 GT/s的link speed的Downstream port，针对该port下连接的device，系统软件必须在其退出fundamental reset之后，至少等待100ms，才向其发送configuration request
- 对于支持大于5.0 GT/s的link speed的Downstream Port，针对该port下连接的device，系统软件必须在其退出fundamental reset之后，至少等待100ms，才向其发送configuration request。软件可以通过polling Data Link Layer Link Active bit，或者通过配置相应的中断，来得知link training何时完成。
- 对于实现了Readiness Time Reporting Extended Capability，并且汇报其reset time小于100ms的device，允许软件在其汇报的reset time之后，向其发送configuration requests
- 系统必须确保在启动时所有对软件可见的components，都在conventional reset后，可接受的最短的时间之后，便能够接受configuration requests
- 强烈建议仅仅当软件启用了Configuration RRS Software Visibility时，使用100ms的等待时间。否则可能会出现completion timeout / platform timeout，以及处理器的长指令处理会卡住

## Function Level Reset