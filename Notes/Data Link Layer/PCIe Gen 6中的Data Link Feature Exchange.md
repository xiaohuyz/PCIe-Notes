# PCIe Gen 6中的Data Link Feature Exchange
记录一下PCIe Gen 6 中有关Data Link Feature Exchange内容的学习。主要基于PCIe SPEC 6.3 和 7.7.4

对于**支持Flit Mode的port**，以及**支持16 GT/s以上data速率的port**，它们必须支持Data Link Feature Exchange协议。而对于其他类型的port，该协议是可选的。

对于实现了Data Link Feature Exchange协议的ports，如果是**Downstream Ports**，那么它必须包含**Data Link Feature Extended Capability，**如果是**Upstream Ports**，那么对该capability的实现是**可选**的。

该Capability寄存器如图所示：
- Remote Data Link Feature Supported Valid，指示该port在DL_Feature状态中，已经接收到了一个Data Link Feature DLLP，因此该port的寄存器里Remote Data Link Feature Supported field内的数据是有效的
- Remote Data Link Feature Supported，用于指示Remote Port所支持的Data Link Features
- Data Link Feature Exchange is Enabled，用于data link feature exchange协议的执行。当该bit为1时，DL层的状态机在reset后训练会进入DL_Feature状态
- Local Data Link Feature Supported，用于指示本port所支持的Data Link Features

Data Link Feature Exchange Protocol的作用，在于将自己本地port的Feature Supported信息发送给remote port，同时从remote port获取remote port的Feature Supported信息。

Data Link Feature Exchange Protocol必须遵循的一些规则如下：
- 当DL层状态机**才进入**DL_Feature状态时： 	
  - 允许清除port的**Data Link Feature Extended Capability**寄存器内的Remote Data Link Feature Supported field和Remote Data Link Feature Supported Valid bit。这是由于训练才进入FL_Feature状态，还没有收到Data Link Feature DLLP，因此对于remote port的功能情况还不知道。
  
- 当DL层处于DL_Feature状态时： 
  - 本port的TL层必须阻止任何TLP的传输，即不允许TLP从TL层向下发送到DL层
  - 发送Data Link Feature DLLP 	
    - Data Link Feature DLLP如图所示 		
    - Feature Supported field，实际上传输的就是本port的寄存器中的Local Data Link Feature Supported Field
    - Feature Ack bit，实际上使用的就是本port寄存器中的Remote Data Link Feature Supported Valid
  - Data Link Feature DLLP每34us至少发送一次。其中物理层状态机LTSSM在Recovery和Configuration状态的时间不计算在内。
  - 当接收到Data Link Feature DLLP时： 	
    - 如果Remote Data Link Feature Supported Valid bit是clear，将DLLP中Feature Supported field的值存入本port寄存器的Remote Data Link Feature Supported field，而后将寄存器中的Remote Data Link Feature Supported Valid bit set
  
- 如果遇到以下情况之一，则退出DL_Feature： 
  - port接收到一个InitFC1 DLLP
  
  - 接收到一个MR-IOV MRInit DLLP
  
  - 在DL_Feature状态，至少接收到一个Data Link Feature DLLP，其中它的Feature Ack bit是set的
  

Data link feature，代表了一个protocol feature，可能存在3种情况，即activated / not activated / not supported。当Remote Data Link Feature Supported Valid bit为**set**，而且某个protocol feature bit在Local Data Link Feature Supported Field和Remote Data Link Feature Supported Field中都是set状态，那么该protocol feature则被视为activated

Data link parameter，即data link的参数，是link两端的port用来交换一些数据用的

- Scale Flow Control，是个feature，port两边都支持，则开启
- Immediate Readiness
- Extended VC Count，指示sending port支持多少VC。**仅仅在Flit Mode中有效，在Non Flit Mode中必须为0**
- L0p Exit Latency，指示sending port的L0p Exit Latency。**仅仅在Flit Mode中有效，在Non Flit Mode中必须为0**