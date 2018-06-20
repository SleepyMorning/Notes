# Android系统电话

Android系统电话从上到下分为以下几层：
- 应用层，包括Dialer、Contacts、InCallUI等
- Java框架层，包括telecomm，telephony
- 平台适配层RIL
- 硬件驱动层

关注点：
- **每层的实现方式**
- **每层之间的通信方式**

1. 平台适配层RIL实现为一个独立的开机自启动的deamon进程，通过AT（qmi或smi）信令与硬件驱动层进行交互；
2. Java框架层实现了RIL.java，通过LocalSocket方式与平台适配层RIL进行交互；
3. 应用层的各种应用则使用binder机制与Java框架层进行交互；
