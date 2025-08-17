# Link宽度切换流程

## 降Lane

1. 和Speed Change类似，只能由Upstream Port发起Link Width Change
2. Upstream Port进入Recovery.RcvrLock，发送TS1 OS，注意，其中speed change为0b
3. Downstream Port返回TS1 OS，注意，其中speed change同样为0b
4. Upstream接收到足够的返回的TS1 OS后，进入Recovery.RcvrCfg，开始发送TS2 OS
5. Downstream Port接收到TS2 OS，进入Recovery.RcvrCfg，开始发送TS2 OS
6. 注意上诉的TS1 OS和TS2 OS中，link和lane number都不变，即和上次训练成功时的一样
7. 两端互传足够数量的TS2 OS之后，都进入Recovery.Idle。此时，Downstream Port开始发送idle symbols，但是Upstream Port被upper direct to Configuration.LinkWidth.Start。发送TS1 OS，其中link / lane number为PAD
8. Downstream Port接收到TS1 OS，发现以前已经配置好的lane，现在接收到TS1 OS，但是lane number却为PAD，则进入Configuration.LinkWidth.Start，开始发送TS1 OS，注意TS1 OS中link number和上次训练结果一样，但是lane number为PAD。
9. Upstream Port接收到这些TS1 OS，进入Configuration.LinkWidth.Accept，返回TS1 OS。但是和DSP不同的是，虽然lane number也是PAD，但是USP会在自己想要inactive的lanes上，发送TS1 OS，其中Link number同样为PAD。
10. Downstream Port接收到这些TS1 OS，进入Configuration.LinkWidth.Accept。发送TS1 OS，对于新训练中仍然active的lanes，link number不变而lane number重新分配号码。而在新训练中inactive的lanes，则link / lane number都为PAD
11. DSP不会停留在Configuration.LinkWidth.Accept太久，它会进入Configuration.Lanenum.Wait
12. Upstream Port接收到这些TS1 OS，而后返回同样的TS1 OS，进入Configuration.Lanenum.Wait
13. DSP收到返回的TS1 OS（Non-PAD / Non-PAD），进入Configuration.Lanenum.Accept。继续在active lanes发送TS1 OS，而将inactive lanes置为EI
14. USP接收到TS1 OS，进入Configuration.Lanenum.Accept。同样返回TS1 OS，并且将inactive lanes设置为EI
15. DSP接收到返回的TS1 OS，则进入Configuration.Complete，开始发送TS2 OS
16. USP接收到到TS2 OS，则进入Configuration.Complete，开始发送TS2 OS
17. DSP接收到足够的TS2 OS，则进入Configuration.Idle，开始发送logical Idle。EP返回同样的Logical Idle。
18. 最后link进入L0