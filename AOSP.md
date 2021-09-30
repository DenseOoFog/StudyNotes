# AOSP
Android官网: https://source.android.com/  
Android Github镜像: https://github.com/aosp-mirror

## 一、Android 系统架构
参考文档：https://developer.android.google.cn/guide/platform
1. Linux Kernel (Linux内核)
  - Android平台基础是Linux内核。Android Runtime(ART)依靠Linux内核来执行底层功能,例如线程和低层内存管理
  - 使用Linux内核可让Android利用主要安全功能，并且允许设备制造商为著名的内核开发硬件驱动程序。  
2. Hardware Abstraction Layer (HAL-硬件抽象层)
  - 硬件抽象层提供标准界面，向更高级别的Java API 框架显示设备硬件功能。HAL包含多个库模块，其中每个模块都为特定类型的硬件组件实现一个界面，例如相机或蓝牙模块。当框架API要方位设备硬件时，Android系统将为该硬件组件加载库模块。
3. Native C/C++ Libraries  / Android Runtime (第三方的C++库 + Android的运行库)  
  对于运行Android5.0(API级别21)或更高版本的设备，每个应用都在其自己的进程运行，并且有其自己的Android Runtime(ART)实例。ART编写为通过执行DEX文件在低内存设备上运行多个虚拟机，DEX文件是一种专为Android设计的字节码格式，经过优化，使用的内存很少。编译工具链(例如**Jack**)将Java源代码编译为DEX字节码，使其可在Android平台上 运行。  
  Android Runtime的部分主要功能包括：
    - 预先(AOT)和即时(JIT)编译
    - 优化的垃圾回收(GC)
    - 在Android 9(API级别28)及更高版本的系统中，支持将应用软件包中的Dalvik Executable格式(DEX)文件转换为更紧凑的**机器代码**。
    - 更好的调试支持，包括专用采样分析器、详细的诊断异常和崩溃报告，并且能够设置观察点以监控特定字段  
    在Android版本5.0(API级别21)之前，Dalvik是Android Runtime。如果您的应用在ART上运行效果很好，那么它应该也可在Dalvik上运行，但反过来不一定。
    Android还包含一套核心运行时库，可提供Java API框架所使用的Java变成语言中的大部分功能，包括一些Java 8**语言功能**
  Native C/C++ Libraries主要功能包括：
    - 许多核心Android系统组件和服务(ART和HAL)构建自原生代码，需要以C和C++编写的原生库。Android平台提供Java框架API以向应用显示其中部分原生库的功能。例如，您可以通过Android框架的Java OpenGL API访问OpenGL ES，以支持在应用中绘制和操作2D和3D图形。 如果开发的是需要C或C++代码的应用，可以使用Android NDK直接从原生代码访问某些原生平台库

4. Java API Framework (Java API 框架)
   您可通过以Java语言编写的API使用Android OS的整个功能集。这些API形成创建Android应用所需的构建块，它们可简化核心模块化系统组件和服务的重复使用，包括以下组件和服务：
    - 丰富、可扩展的**视图系统**，可用以构建应用的UI,包括列表、网格、文本框、按钮甚至可嵌入的网络浏览器
    - **资源管理器**，用于访问非代码资源，例如本地化的字符串、图形和布局文件
    - **通知管理器**，可让所有应用在状态栏中显示自定义提醒
    - **Activity管理器**，用于管理应用的生命周期，提供常见的**导航返回栈**
    - **内容提供程序**，可让应用访问其他应用(例如“联系人”应用)中的数据或者共享其自己的数据  
    开发者可以完全访问Android系统应用使用的框架API
5. System Apps (系统应用)  
   Android随附一套用于电子邮件、短信、日历、互联网浏览和联系人等的核心应用。平台随附的应用于用户可以选择安装的应用一样，没有特殊状态。因此第三方应用可成为用户的默认网络浏览器、短信Messenger甚至默认键盘(有一些例外，例如系统的“设置”应用)。
   系统应用可用作用户的应用，以及提供开发者可从其自己的应用访问的主要功能。例如，如果您的应用要发短信，您无需自己构建该功能，可以改为调用已安装的短信应用向您指定的接受者发送消息。

为了更深入地掌握Android整个架构思想以及各个模块在Android系统所处的地位与价值，计划以Android系统启动过程为主线，以进程的视角来诠释Android M系统全貌，全方位的深度剖析各个模块功能，争取各个击破。

## 二、通用系统映像(GSI)
  参考文档：https://developer.android.google.cn/topic/generic-system-image
  通用系统映像(GSI)是一种纯Android实现，采用未经修改的Android开源项目(AOSP)代码，可在各种Android设备上运行。  
  您可以比以往更早开始在更大范围内测试应用：
  - 使测试覆盖到更多的现实设备
  - 有更多时间在解决应用兼容性问题
  - 有更多机会来解决应用开发者报告的与Android操作系统不兼容的问题  

GSI项目可以在下一个操作系统版本发布之前，提供更多方法来提高应用和操作系统的质量，从而帮助改善Android生态系统。GSI项目是**开源**。
GSI中包含所有搭载Android 9及更高版本的设备中的核心系统功能。换句话说，GSI中不包含设备制造商的定制。但在以下情况下，您可能会遇到行为差异：
  - 涉及界面的互动。
  - 需要更新的硬件功能的工作流。

检查设备合规性  
GSI仅适用于具有以下特征的设备：
  - 引导加载程序已解锁
  - 完全符合Treble要求
  - 出厂时搭载Android 9 (API级别28)或更高版本。从较低版本升级到Android9的设备不一定支持GSI
   如需确定设备是否可以使用GSI以及应该安装哪个GIS操作系统版本，请执行以下操作：
  1. 运行以下命令来检查设备是否支持Treble，如果响应为false，表示设备不兼容GSI
```
      adb shell getprop ro.treble.enabled
```    

  2. 运行以下命令来检查设备是否支持跨版本安装：
```
      adb shell cat /system/etc/ld.config.version_identifier.txt \ |grep -A 20 "\[vendor\]"
```
在输出的[vendor]部分中查找namespace.default.isolated如果该属性的值为true，表示设备完全支持**供应商原生开发套件**(VNDK)，因此可以使用比设备端操作系统版本更高的任何GSI操作系统(OS)版本。如果该属性的值为false，表示不完全兼容VNDK，因此只能使用与设备端操作系统版本相同的GSI。例如，如果搭载Android9的设备与VNDK不兼容，则只能加载Android GSI映像。

  3. GSI CPU架构类型必须与设备的CPU架构匹配。如需为GSI映像查找合适的CPU架构，请运行以下命令：
```
    adb shell getprop ro.product.cpu.abi
```
通过该输出确定在刷鞋设备时要使用的GSI映像。例如，在Pixel3上，输出会指明CPU架构是arm64-v8a,因此您需要使用arm64类型的GSI

## 三、动态系统更新 (DSU)
动态系统更新(DSU)是Android 10中引入的一项系统功能，可执行以下操作：
  - 将新的GSI(或其他Android系统映像)下载到您的设备上
  - 创建新的动态分区
  - 将下载的GSI加载到新的分区
  - 在设备上将GSI作为来宾操作系统启动
DSU还可让您在当前系统映像和GSI之间轻松切换，因此您在使用GSI时不会面临当前系统映像受损的风险。

DSU依赖于Android动态分区功能，并要求GSI作为可信系统映像由Google或您的OEM进行签名
DSU是设备制造商提供的功能，从Android10 Beta4开始，Google在Pixel 3和更高版本的设备上启用了DSU
  - 在使用userdebug Android build的设备上，您可以在设置->系统->开发者选项->功能标记->settings_dynamic_system中启用该功能。
  - 在其他设备上,请使用一下adb命令
```
  adb shell setprop persist.sys.fflag.override.setttings_dynamic_system true
```

## 四、Android架构(进程视角)
Android系统启动过程由上图从下往上的一个过程是由Boot Loader引导开机，然后依次进入->Kernel->Native->Framework->App
关于Loader层：Boot ROM 引导芯片开始从固化的ROM里的预设代码开始执行，然后加载引导程序到RAM,
Boot Loader这是启动Android系统之前的引导程序，主要是检查RAM,初始化硬件参数等功能。
1. Linux内核层
启动Kernel的swapper进程(pid = 0):该进程又称为idle进程，系统初始化过程kernel由无到由开创的第一个进程，用于初始化进程管理、内存管理，加载Display,Camera Driver,Binder Driver等相关工作
启动Kthreadd进程(pid = 2):是Linux系统的内核进程，会创建内核工程线程kworkder，软中断线程ksoftirqd,thermal等内核守护进程。kthreadd进程是所有内核进程的鼻祖。
2. 硬件抽象层
硬件抽象层(HAL)提供标准接口，HAL包含很多库模块，其中每个模块都为特定类型的硬件组件实现一组接口，比如WIFI/蓝牙模块，当框架API请求访问设备硬件时，Android系统将为该硬件加载响应的库模块
3. Android Runtime & 系统库
每个应用都在其自己的进程中运行，都有自己的虚拟机实例。ART通过执行DEX文件可在设备运行多个虚拟机，DEX文件是一种专为Android设计的字节码格式文件，经过优化，使用内存很少。ART主要功能包括：预先(AOT)和即时(JIT)编译，优化的垃圾回收(GC)，以及调试相关的支持。
这里的Native系统库主要包括init孵化来的用户空间的守护进程、HAL层以及开机动画等。启动init进程(pid = 1)，是linux系统的用户进程，init进程是所有用户进程的鼻祖。
  - init进程会孵化出ueventd、logd、healthd、installd、adbd、Imkd等用户守护进程
  - init进程还启动servicemanager(binder服务管家)、bootanim(开机动画)等重要服务
  - init进程孵化出Zygote进程,Zygote进程是Android系统的第一个Java进程(即虚拟机进程),Zygote是所有Java进程的父进程，Zygote进程本身是由init进程孵化而来的。
4. Framework层
  - Zygote进程,是由init进程通过解析init.rc文件后fork生成的，Zygote进程主要包含:
  - 加载ZygoteInit类，注册Zygote Socket服务端套接字
  - 加载虚拟机
  - 提前加载类preloadClasses
  - 提前加载资源preloadResouces
  - System Server进程，是由Zygote进程fork而来，System Server是Zygote孵化的第一个进程，System Server负责启动和管理整个Java framework，包含ActivityManager,WindowManager,PackageManager,PowerManager等服务。
  - Media Server进程，是由init进程fork而来，负责启动和管理整个C++ framework,包含AudioFlinger,Camera Service等服务。
5. App层
  - Zygote进程孵化出的第一个App进程是Launcher,这是用户看到的桌面App。
  - Zygote进程还会创建Browser,Phone,Email等App进程，每个App至少运行在一个进程上。
  - 所有的App进程都是由Zygote进程fork生成的。
6. Syscall && JNI
  - Native与Kernel之间有一层系统调用(SysCall)层，见**Linux系统调用(Syscall)原理**。
  - Java层与Native(C/C++)层之间的纽带JNI,见**Android JNI原理分析**。

















































1
