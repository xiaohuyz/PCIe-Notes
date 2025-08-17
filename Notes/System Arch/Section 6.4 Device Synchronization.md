# Section 6.4 Device Synchronization

系统软件需要一个“停止”机制，从而保证对于系统内的一个device而言，没有正在运行的transaction。例如，如果没有这种“停止”机制，那么在系统运行时，对于Bus Numbers的renumbering会使得一个device，当它的某些Requests或者Completions还在传输时，它的Requester ID发生改变，这种。