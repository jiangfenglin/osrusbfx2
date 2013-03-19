# **Step by Step, 为OSRFX2创建一个KMDF驱动程序**

## ***Chapter 1. 基础知识***

### 1.1 Windows驱动的一些基本知识
在进入WDF之前，我们有必要先回顾并深刻了解Windows这个操作系统和驱动相关的一些基本知识。WDF虽然很抽象很高级，但拨开其面纱，内部仍然是万变不离其宗，并没有离WDM太远。

### 1.1.1 Windows系统架构
Windows作为一个分层的系统架构，从上到下依次包括：
应用程序（Application）
Windows API，可以主要理解为Win32 API。运行在用户态的应用程序不可以直接访问内核态的模组，但它们可以通过调用Win32 API发起I/O请求。
内核子系统（Kernel Subsystems），Windows内核的主要组成部分，它们提供驱动开发接口DDI供驱动调用。在Windows的众多内核子系统中，驱动主要和以下三个子系统打交道。

	- I/O管理器（I/O Manager），主要责任是将应用程序发起的I/O请求发给对应的驱动或者反之将驱动从物理设备收到的应答返回给应用程序。
	- 即插即用管理器（PnP Manager），处理设备插拔事件，还要负责构造或者析构设备栈（device stack）。
	- 电源管理器（Power Manager），处理PC的电源状态管理，比如休眠，恢复等等。

![Windows系统架构](./images/DDMWDF-2-1.PNG)

### 1.1.2 设备对象和设备栈
在Windows系统中，一个物理设备要正常工作，往往需要多个驱动的支持。PnP管理器检测到一个物理设备连接到系统总线并且对应的驱动也找到后，就会在系统中按照栈的形式从上到下将这些驱动组织起来。为每一个驱动对应地创建一个设备对象，这样，当I/O管理器从应用程序收到一个I/O请求时，会按照这个栈的顺序将I/O请求依次向下转发给各级设备对象，直到在某一层被处理结束。设备对象作为I/O管理器和驱动程序之间的代理，其数据结构中包含了驱动注册的回调函数指针集合。I/O管理器通过这些回调函数和我们的驱动交互，调用我们驱动实现的功能函数。
考虑到一台PC同时可能会处理多个相同的设备，所以我们的驱动代码要考虑同时支持多个设备栈。但每个设备栈只对应着连在系统上的一个物理设备。
![设备对象和设备栈](./images/DDMWDF-2-2.PNG)

以我们即将介绍的OSRFX2设备和驱动为例。


### 1.1.3 设备树




### 1.2 WDF简介

Windows Driver Foundation (WDF)是Windows上开发驱动的最新架构。基于这个架构我们可以开发用户态的驱动程序（UMDF），也可以开发内核态的驱动程序（KMDF）。

WDF为各种各样的设备类型抽象了一个统一的驱动模型，这个模型的最最重要的特性可以概括为三个主要的关键模型 - 对象模型(Object Model)；I/O模型(I/O Model)以及PnP/Power模型(PnP/Power Model)。之所以称其为统一，还有一方面的意思是，如果我们基于WDF来开发驱动，那么规范是已经定义好的，所有的WDF驱动都要在这个框架下根据WDF定义的接口和行为操作，否则就不能享受WDF给我们带来的福利。
![WDF模型](./images/DDMWDF-4-1.PNG)


#### 1.2.1. WDF提供了一个设计良好的对象模型

WDF将Windows的内核数据结构和与驱动操作相关的数据结构封装为对象，例如设备（device）,驱动（driver）, I/O请求（I/O request）, 队列（queue）等等。这些对象有些是由FrameWork创建并传递给WDF驱动使用，有些对象可以由驱动代码根据自己的需要创建，使用并删除。我们驱动要做的一件很重要的事情就是要学会和这些WDF对象打交道并把他们组织起来为实现我们设备的特定功能服务。

##### 1.2.1.1. 对象的编程接口

	* 每个对象有属性(Properties),对属相的访问要通过方法(Methods)。
	* 每个对象提供了操作方法(Methods)，WDF驱动通过调用这些函数操作对象的属性或者指示对象动作。  
	* 每个对象支持事件 (Events)。实现上是回调函数。FrameWork提供了对这些事件的缺省处理，我们的驱动不需要去关心所有的回调函数，但是如果我们的驱动希望对某个事件添加自己的特定于设备的处理时（这总是需要的），则可以注册这个事件的回调函数。这样当系统状态改变或者一个I/O请求发生时FrameWork就会调用驱动注册的回调函数，并执行我们驱动自己的代码，可以类比于C++中的重载一样。  

##### 1.2.1.2 在对象模型中提供了一套设计精巧的对象依附关系
这套关系可以帮助我们的驱动代码简化对对象生命周期的管理，具体来说，如果一个对象B是依附于等级A，那么当A被删除时，FrameWork会自动删除对象B。就像封建社会的主人和仆从的关系，主子完蛋的时候，仆从也要跟着一起玩完。  

	* Driver对象的级别最高，是WDF对象这棵大树的根，也是FrameWork为我们的驱动创建的第一个对象  
	* 有些对象必须依附于Device对象，或者是Device对象的仆从对象的仆从。比如Queue对象，都依附于Device而存在。  
	* 有些对象可以有多个主子，比如Memory对象。

##### 1.2.1.3 对象的上下文空间
FrameWork模型中的对象的数据结构都是由WDF定义好的，驱动如果为了自己设备的需要想要扩展这些对象存储的数据，可以为对象增加上下文空间。FrameWork对象数据结构中有一个域存放了指向这个上下文空间的句柄。

#### 1.2.2. WDF定义了一个统一的I/O模型

WDF的I/O模型负责管理I/O请求（I/O Request）。I/O请求缓存在I/O队列（I/O Queue）里，如果本驱动无法处理完该I/O请求则它还要继续发送给下一个处理者-I/O目标(I/O Target)-继续处理直到完成。所以WDF的I/O模型颠来倒去就是围绕着I/O请求，I/O队列和I/O目标这三个主要组成部分来介绍。当然幕后还是不要忘记了FrameWork这个大老板。

FrameWork做为OS和驱动之间的一个中间层，当Windows向WDF驱动发送I/O请求时，FrameWork首先收到该请求，并决定是由FrameWork自己处理还是调用驱动注册的回调函数进行处理。如果需要交给驱动处理，则FrameWork就要进入利用一套机制代表驱动进行I/O请求的分发，排队，结束和取消操作。  

请注意这里FrameWork并不是简单地将I/O请求全权交给驱动进行处理的，而是通过内建了一套完善的PnP和Power事件处理以及状态管理机制来跟踪所有I/O请求，并在合适的时间点上呼叫驱动注册的回调处理函数。这些时间点包括系统电源状态的变化（比如进入休眠，从休眠退出）；设备和系统之间的连接发生变化（比如设备连上主机，或者设备从主机上断开）等等。这么做可以极大地减轻原来驱动处理的负担，并代替驱动处理好了复杂的同步处理。  

不需要跟踪管理复杂的I/O状态、事件,也不需要太多顾虑事件的同步，基于WDF的驱动要做的事情就只剩下创建合适的队列和注册定义在队列上的事件处理回调函数并加入自己的处理代码就可以了。  

##### 1.2.2.1. I/O队列（Queue）
所有的I/O请求在被分发给驱动之前都需要进入队列。队列对象是依附于设备对象而存在的。FrameWork负责管理队列，而驱动代码负责创建队列并设置队列的属性告诉FrameWork该如何管理队列对象。
WDF驱动创建队列时需要对队列进行仔细的设置，这样FrameWork才会更好地满足我们驱动的需要来为我们服务。对于驱动创建的每一个队列对象，我们要配置好：  

	* 该队列可以处理的I/O请求的类型。  
	* FrameWork应该怎样将队列上的I/O请求分发给WDF驱动。Parallel，Sequential还是Manual。  
	* 当PnP和Power事件发生时FrameWork应该如何处理队列，包括启动，停止还是恢复队列。  如果我们希望FrameWork帮助我们管理影响队列的电源状态变化事件，那么我们就配置这个队列是使能电源管理的。只要这样FrameWork就会为我们检测电源状态变化，并在设备进入工作状态时才会为其分发消息，此时队列自动开始工作；当设备退出工作状态时，FrameWork就让队列停止工作，不会为该队列分发消息了。这么做的好处就是驱动如果知道这个队列是使能了电源管理的，则收到I/O请求时就不用再去确认设备的状态了。当然我们也可以不让FrameWork为我们的队列管理电源状态变化。更详细的描述可以参考DDMWDF第7章"Plug and Play and Power Management Support in WDF"。

##### 1.2.2.2. I/O Request
I/O请求有三种类型：包括Read请求，Write请求和Device I/O Control请求。

I/O Request Cancellation
Windows的I/O处理在内核中都是异步的，所以在没有WDF之前驱动代码要处理好取消事件不是件容易的事情，驱动需要自己定义和管理至少一个或者多个锁来处理潜在的竞争条件。在WDF中FrameWork代替驱动做了这些工作，为驱动创建的队列提供了内建的缺省锁机制来处理I/O请求的取消事件，FrameWork不仅可以代替驱动取消那些已进入队列排队但还没有分发给驱动的事件，对于那些已经分发的请求，只要驱动设置允许，FrameWork也可以将其取消。

##### 1.2.2.3. I/O Targets
I/O目标对象从字面上理解就是可以接收I/O请求的对象，具体来说它代表了接收请求的运行在内核态的驱动实体。  

从I/O目标代表的驱动对象距离发起I/O请求对象的远近，我们可以将其分为缺省I/O目标对象和远距离I/O目标对象。 
 
缺省I/O目标：紧挨着发起I/O请求对象的设备栈中的下一级驱动。当我们调用FrameWork的API创建一个设备对象时，FrameWork为我们创建和初始化该设备对象缺省的I/O目标对象，该缺省I/O目标对象被定义为设备对象的子对象并保存在该设备对象下。我们可以调用FrameWork的API来得到I/O目标对象的句柄。FrameWork仅仅为功能驱动和过滤驱动对象创建缺省的I/O目标对象。

除了发起请求的对象和紧挨着发起I/O请求对象的设备栈中的下一级驱动对象外的目标对象都统称为远距离I/O目标对象。远距离I/O目标对象和发起请求的对象可能在一个设备栈中也可能不在一个设备栈中。  

除了将I/O目标对象根据距离远近分为缺省和远距离两种类型以外，根据目标对象代表的设备类型来分还可以分为通用目标对象和专有目标对象两种。

通用I/O目标可以用来代表所有的设备类型，驱动使用它们时不需要对其格式进行特别的处理。而专有I/O目标的格式则和特定的某一类设备类型相关。USB目标对象就是WDF定义的一种专有I/O目标对象，USB目标对象传送的请求使用WDF定义的URB（USB request blocks）格式，WDF提供专用的API来操作这些URB数据。随着WDF支持越来越多新的设备类型，则WDF也会扩展并定义越来越多的新的专有I/O目标类和操作他们的API。


#### 1.2.3. PnP和Power管理模型-状态机
提供了内建的对PnP和Power的管理，对状态的管理和迁移提供了缺省操作。  
WDF驱动不需要自己实现一个复杂的状态机来跟踪PnP和电源状态来确定当状态发生变化时该采用什么动作。FrameWork自己内部有一个状态机为我们做了这一切并定义了需要实现的回调函数入口。驱动只要对自己关心的事件注册这些回调函数并在实现中加入对自己设备特定的处理，对其他的事件完全可以交给FrameWork去处理就好了。


## ***Chapter 2. OSRFX2 - Step By Step***
WDK中的OSRFX2例子在WDF出现之后已经基于WDF完全重写了。我们这里正是基于这个例子来循序渐进地了解一下如何开发一个基于WDF的KMDF驱动，以及在这个过程中我们需要注意哪些重要的知识点。

### 2.1。 Step1 - 创建一个最简单的KMDF功能驱动。

一个最小化的KMDF驱动可以只实现**DriverEntry**和**EvtDeviceAdd**这两个回调函数。WDF框架提供了其他所有缺省的功能，使之可以支持加载驱动并响应PNP和电源事件。我们可以安装，卸载，使能，去能该驱动，同时在OS挂起和恢复情况下该最小驱动也可以正常工作。

#### 2.1.1. 驱动入口点[DriverEntry]

当一个驱动被加载到内存后第一个被FrameWork调用的函数，最为驱动程序的主入口，类似于用户态应用进程的main函数。根据WDF的定义我们需要在这个函数中执行创建驱动对象的动作。
对标准驱动来说，这个函数是非常简单的：FrameWork在调用这个函数时传入两个参数，参数一DriverObject是指向WDF封装的WDM的Driver对象的指针，该内存是由内核的IO Manager分配的，我们需要在DriverEntry中对它进行初始化。参数二RegistryPath给出了内核在系统注册表中为我们的驱动分配的单元的路径，我们可以在这个路径下去存储一些驱动相关的信息。这里我们没有使用第二个参数。

首先我们要调用`WDF_DRIVER_CONFIG_INIT`来初始化驱动对象的配置结构。`WDF_DRIVER_CONFIG`结构中包含有两个回调函数，我们这里用到了其中一个[EvtDriverDeviceAdd]。每个基于WDF实现的并支持即插即用的驱动都必须实现该回调函数并确保设置好`WDF_DRIVER_CONFIG`中该回调函数的地址，这样在设备连接并被系统枚举时该回调函数才会被系统正确调用到。  
`WDF_DRIVER_CONFIG       config;`  
`WDF_DRIVER_CONFIG_INIT(&config, EvtDeviceAdd);`  
初始化配置（configuration）和属性（attribute）之后(Step1里对属性没有特别的处理，直接使用缺省属性设置)，我们就可以调用 [WdfDriverCreate]为正在被系统加载的驱动创建一个驱动对象，我们不需要自己申请内存，因为正如前面所述，驱动对象实体已经被FrameWork创建好并给出了其指针句柄。我们的任务就是根据我们的需要对其初始化（虽然函数的名字叫create）。  
`status = WdfDriverCreate(DriverObject,`  
`                         RegistryPath,`  
`                         WDF_NO_OBJECT_ATTRIBUTES,`   
`                         &config,`  
`                         WDF_NO_HANDLE`  
`                        );`  

#### 2.1.2 添加新设备入口点[EvtDriverDeviceAdd]

每当PnP Manager检测到一个新设备，系统都会调用EvtDeviceAdd函数。我们要做的工作就是在此函数中创建并初始化设备对象。

FrameWorks在调用该回调函数时传入两个参数，参数一是驱动对象的句柄，这个应该就是我们在DriverEntry里创建的驱动对象。另一个参数是指向`WDFDEVICE_INIT`的指针句柄DeviceInit，该句柄是由FrameWork创建的，如果我们不需要对其进行修改，直接传递给[WdfDeviceCreate]

在创建和初始化设备对象时我们用到了一个重要的WDF API [WdfDeviceCreate]。调用WdfDeviceCreate会创建一个框架设备对象(framework device object)来表达一个功能设备对象(functional device object, 简称FDO)或者是一个物理设备对象(physical device object, 简称PDO)。我们这里创建的OSRFX2设备是一个FDO.

在调用WdfDeviceCreate之前，我们根据需要初始化WDFDEVICE_INIT结构对象DeviceInit。在Step1这里我们接受缺省对象，所以什么都不用做。  
Step1中创建设备对象时我们也没有设置对象属性 - `WDF_NO_OBJECT_ATTRIBUTES`, 在Step2中我们可以看到对属性更多的设置。  
`status = WdfDeviceCreate(&DeviceInit, WDF_NO_OBJECT_ATTRIBUTES, &device);`

### 2.2. Step2 - 准备设备对象
参考DDMWDF Chapter6 - Queues and Other Support Objects的一段话

主要包括:

- 为OSRFX2注册设备接口([device interface])
- 为OSRFX2添加上下文
- 为OSRFX2准备硬件IO

#### 2.2.1. 为OSRFX2注册设备接口([device interface])
KMDF驱动程序运行在内核态，为了让用户态的应用程序APP能够访问我们的驱动程序对象，主要是设备接口对象，驱动程序需要和操作系统协作在暴露出应用程序可以在用户态访问的接口，这就是[device interface]，俗称符号链接(symbolic link)。应用程序可以将设备接口的符号链接作为调用Win32 API的参数来和驱动打交道。更详细的描述可以参考[Using Device Interfaces]。

具体的操作可分如下几部：
首先要用[guidgen.exe]为OSRFX2的接口对象类产生一个GUID:  
`DEFINE_GUID(GUID_DEVINTERFACE_OSRUSBFX2, // Generated using guidgen.exe`  
`   0x573e8c73, 0xcb4, 0x4471, 0xa1, 0xbf, 0xfa, 0xb2, 0x6c, 0x31, 0xd3, 0x84);`  
`// {573E8C73-0CB4-4471-A1BF-FAB26C31D384}`  

然后在EvtDeviceAdd回调函数中调用[WdfDeviceCreateDeviceInterface]即可。  
`status = WdfDeviceCreateDeviceInterface(device,(LPGUID) &GUID_DEVINTERFACE_OSRUSBFX2,NULL);// Reference String`  
一旦返回成功则用户态的应用程序就可以用我们注册的设备接口类来查找并访问我们的设备了。  
为更好地理解从上到下I/O设备的访问路径，我们可以再看一下osrfx2的测试程序是如何访问设备的。
参考\osrusbfx2\kmdf\exe\testapp.c文件的GetDevicePath函数和OpenDevice函数。大致的流程如下：  
	- 应用程序调用[SetupDiGetClassDevs]，得到所有属于`GUID_DEVINTERFACE_OSRUSBFX2`设备接口类的设备信息集合([device information sets])。  
	- 调用[SetupDiEnumDeviceInterfaces]从设备信息集合中找到第一个该类型设备的接口句柄。这里我们只处理一个设备所以只遍历第一个检测到的设备。  
	- 得到设备接口句柄后我们就可以调用[SetupDiGetDeviceInterfaceDetail]来获得设备文件的内核路径。    
	- 拿到设备文件路径后就可以调用[CreateFile]，CreateFile一旦成功该函数会返回打开文件的句柄，应用程序自此可以使用该文件句柄向设备发送I/O请求。  

#### 2.2.2 为OSRFX2添加上下文
Step1中创建的设备对象采用的都是缺省配置，在实际使用中我们总是需要为我们自己的设备添加特定的操作的。在此过程中我们很可能需要为了我们的操作逻辑添加一些存储对象。比如在Step2中我们需要向OSRFX2发送我们自己的I/O控制命令就需要配置OSRFX2自己的Target对象并将其记录下来并在FrameWork设备对象的整个生命周期中都需要访问。为此我们可以为我们驱动的设备对象创建自己的上下文空间。我们可以简单地将上下文空间理解为一块不可分页的内存区域。我们可以将自己驱动程序需要的一些变量定义在这个上下文数据区中，创建好之后我们可以通知WDF将其附加到WDF管理的框架对象中，以便在框架对象的生命周期中访问。更详细的内容我们可以参考[Framework Object Context Space].
这里以OSRFX2为例我们看一下为设备对象创建上下文的过程。

首先我们要自定义一个数据结构DEVICE_CONTEXT.  
`typedef struct _DEVICE_CONTEXT {`  
`  WDFUSBDEVICE      UsbDevice;`  
`  WDFUSBINTERFACE   UsbInterface;`  
`} DEVICE_CONTEXT, *PDEVICE_CONTEXT;`  
在设备上下文结构中我们定义了两个面向底层硬件的WDF对象。具体这两个对象是如何创建的请参考“为OSRFX2准备硬件IO”。  
结构定义好后我们还要调用一个WDF的宏`WDF_DECLARE_CONTEXT_TYPE_WITH_NAME(DEVICE_CONTEXT, GetDeviceContext)`,第一个参数是上面我们定义的上下文数据结构的名字，第二个参数是给出我们期望访问上下文数据结构的函数名称GetDeviceContext。这样定义后，在代码里我们就可以调用GetDeviceContext来得到指向上下文对象的指针了。WDF如此这般主要目的是要封装WDFDEVICE对象，避免我们直接去访问其数据结构。

接着我们需要在调用WdfDeviceCreate创建设备对象时增加对对象属性(`WDF_OBJECT_ATTRIBUTES`)的设置。  
 - 先定义一个属性对象`WDF_OBJECT_ATTRIBUTES attributes;`  
 - 然后通过以下的宏初始化这个属性对象，同时通知FrameWork将我们定义的设备上下文对象(DEVICE_CONTEXT)插入到属性对象结构中。  
`WDF_OBJECT_ATTRIBUTES_INIT_CONTEXT_TYPE(&attributes, DEVICE_CONTEXT);`  
 - 最后在调用WdfDeviceCreate时我们就不会再象Step1中那样采用缺省的属性参数了，将我们先前定义并初始化好的属性对象attributes作为第二个参数传入。这样FrameWork就知道需要将我们的上下文对象和设备对象建立起联系。
`status = WdfDeviceCreate(&DeviceInit, &attributes, &device);`

#### 2.2.3 为OSRFX2准备硬件I/O
FrameWork要求我们在另一个重要的回调函数[EvtDevicePrepareHardware]中对物理设备进行初始化操作以确保驱动程序可以访问硬件设备。比如：发送I/O请求，或者建立物理地址和虚拟地址之间的映射以便驱动可以访问系统分配给设备的地址等。EvtDevicePrepareHardware所进行的初始化操作应该是设备枚举后只执行一次的动作。对于其他需要在设备进入工作状态(D0)的初始化操作，我们建议放在[EvtDeviceD0Entry]中执行。

首先我们来看一下如何注册EvtDevicePrepareHardware。为了向FrameWork注册EvtDevicePrepareHardware，我们需要在[EvtDriverDeviceAdd]中调用[WdfDeviceCreate]之前对[WdfDeviceCreate]的DeviceInit参数进行初始化,主要是调用[WdfDeviceInitSetPnpPowerEventCallbacks]对DeviceInit设置与PnP以及电源管理相关的回调函数。示例代码如下：  
`WDF_PNPPOWER_EVENT_CALLBACKS pnpPowerCallbacks;`  
`WDF_PNPPOWER_EVENT_CALLBACKS_INIT(&pnpPowerCallbacks);`  
`pnpPowerCallbacks.EvtDevicePrepareHardware = EvtDevicePrepareHardware;`  
`WdfDeviceInitSetPnpPowerEventCallbacks(DeviceInit, &pnpPowerCallbacks);`  

为OSRFX2准备硬件I/O的主要工作就是要在Step1创建FDO的基础上为OSRFX2的FDO创建并关联一系列的[USB I/O Targets]。我们的驱动需要通过这些对象和真正的OSRFX2设备建立USB的会话并开展通讯。  

首先我们要创建一个USB目标对象。调用[WdfUsbTargetDeviceCreate]并将该函数创建的WDFUSBDEVICE对象保存在我们的上下文空间中以备后用。我们可以看到该API的第一个参数是WDFDEVICE。这样我们调用WdfUsbTargetDeviceCreate后WDF就将我们的设备对象FDO和USB对象Target联系起来了。  
`status = WdfUsbTargetDeviceCreate(Device,WDF_NO_OBJECT_ATTRIBUTES,&pDeviceContext->UsbDevice);`  

创建完USB设备对象后我们就需要对其进行配置。在这里我们必须要做的头一件事情就是要对设备选择一个配置(selects a USB configuration).一个配置定义了一个USB设备的一种工作模式，会暴露出几个接口(interfaces)，每个接口有几个端点(endpoints)，每个端点的类型，方向等等。总之只有选择了合适的配置我们才知道这个USB设备是如何工作的，从而我们才知道如何与该USB设备交流，不是吗？  
为了选择配置，我们先定义一个配置参数对象：  
`WDF_USB_DEVICE_SELECT_CONFIG_PARAMS configParams;`  
根据OSRFX2的定义，它是一个单接口的设备，所以我们将其初始化为一个单接口设备：  
`WDF_USB_DEVICE_SELECT_CONFIG_PARAMS_INIT_SINGLE_INTERFACE(&configParams);`  
最后我们将USB设备根据我们的需要选择好并创建USB接口对象并将该接口对象也存储在我们的上下文中备用。  
`status = WdfUsbTargetDeviceSelectConfig(pDeviceContext->UsbDevice,WDF_NO_OBJECT_ATTRIBUTES,&configParams);`  
`pDeviceContext->UsbInterface = configParams.Types.SingleInterface.ConfiguredUsbInterface; `  


### 2.3. Step3 - USB控制传输.

在继续Step3和Step4之前，我们要先理解一些KMDF驱动中处理I/O Requests相关的概念。详细的描述可以参考[Handling I/O Requests in KMDF Drivers]。

[Framework Request Objects] - I/O请求对象，是设计为表示I/O Manager发送给驱动的I/O请求。每个I/O请求封装了一个WDM的I/O Request packet,或者简称为IRP。基于WDF写的驱动一般不需要直接访问IRP结构，而是通过调用[Framework Request Objects]提供的成员函数对其进行操作。

[Framework Queue Objects] - I/O队列对象，是设计为表示用来缓存I/O请求对象的队列。为了简化驱动程序员编写WDM驱动的工作量和处理并行I/O事件，串行I/O事件的复杂性，WDF抽象出了I/O队列的概念来接收驱动需要处理的IO请求。对每个设备可以创建一个或多个队列。FrameWork为I/O队列类定义了一个回调函数的集合，WDF称之为[Request Handlers]。我们可以选取我们部分回调函数并加入我们自己的处理代码（类似C++中虚函数重载的概念）来处理我们感兴趣的事件，FrameWork也为队列类定义了一组操作成员函数供驱动操作队列。

#### *注册I/O控制*

驱动程序的主要面向用户层的接口就是处理IO。用户程序通过调用DeviceIoControl，传入I/O控制码（I/O Control Codes）来通知驱动响应特定的请求。详细的描述可以参考[Using I/O Control Codes]。 定义I/O控制码很简单，有一个Windows定义的宏`CTL_CODE`.  
`#define IOCTL_INDEX                     0x800`  
`#define FILE_DEVICE_OSRUSBFX2          0x65500`  
`#define USBFX2LK_SET_BARGRAPH_DISPLAY 0xD8`  
`#define IOCTL_OSRUSBFX2_SET_BAR_GRAPH_DISPLAY CTL_CODE(FILE_DEVICE_OSRUSBFX2, IOCTL_INDEX + 5, METHOD_BUFFERED, FILE_WRITE_ACCESS)`  
和大多数驱动一样，OSRFX2的驱动在EvtDriverDeviceAdd回调函数中创建自己的I/O队列.  
首先定义一个类型为`WDF_IO_QUEUE_CONFIG`的对象来设置队列的属性特征。  
`WDF_IO_QUEUE_CONFIG                 ioQueueConfig;`  
我们可以根据自己的需要对不同的IO请求创建不同功能的队列来进行处理。在Step3这个例子里为简单起见我们只创建了一个缺省的队列。所谓缺省队列就是说，一旦你为你的设备创建了一个缺省的I/O队列，FrameWork会把所有的I/O请求都放到这个缺省队列里来进行处理，除非你为这个I/O请求创建了其他的处理队列。

初始化缺省队列我们可以调用WDF的宏`WDF_IO_QUEUE_CONFIG_INIT_DEFAULT_QUEUE `。第二个参数传入WdfIoQueueDispatchParallel表明这个队列不会因为我们的I/O处理函数而阻塞。有关I/O分发的机制可以参考[Dispatching Methods for I/O Requests]。  
`WDF_IO_QUEUE_CONFIG_INIT_DEFAULT_QUEUE(&ioQueueConfig, WdfIoQueueDispatchParallel);`  
注册IO Control的回调函数入口[EvtIoDeviceControl]：  
`ioQueueConfig.EvtIoDeviceControl = EvtIoDeviceControl;`  
OK, 接下来我们就可以调用[WdfIoQueueCreate]来创建这个队列了。
`status = WdfIoQueueCreate(device,&ioQueueConfig,WDF_NO_OBJECT_ATTRIBUTES,WDF_NO_HANDLE);`  

以上工作做好后，当应用程序调用DeviceIoControl并传入`IOCTL_OSRUSBFX2_SET_BAR_GRAPH_DISPLAY`来设置LED bar的亮和灭的时候，我们的驱动程序的EvtIoDeviceControl函数就会被FrameWork激活回调。具体的操作就看我们在EvtIoDeviceControl里的处理代码了。因为应用APP在调用DeviceIoControl的时候除了指明IO控制码还需要给定一些其他的参数。对于设置LED bar来说就是要指定点亮或者熄灭8盏灯中的哪一盏。下面我们就来看看用户态的APP和内核态的驱动是怎样传递这些内容的以及我们的驱动又是怎样将这些内容传递给真正的执行体--"设备"的。

#### *处理I/O控制*
现在我们来看一下EvtIoDeviceControl中是如何处理应用发起的`IOCTL_OSRUSBFX2_SET_BAR_GRAPH_DISPLAY`请求的。
[EvtIoDeviceControl]一共有五个参数，都是入(`_In_`)参。因为`IOCTL_OSRUSBFX2_SET_BAR_GRAPH_DISPLAY`操作目的是设置设备的LED bar，所以相对于驱动，没有需要传出给应用的内容，所以没有用到OutputBufferLength；应用有要传给驱动设置LEDbar的内容，大小是一个UCHAR，也就是一个字节，所以InputBufferLength是有效的。我们可以在EvtIoDeviceControl里判断一下，如果太小说明有问题。  
`if(InputBufferLength < sizeof(UCHAR)) {`  
...  
`}`  
接下来我们就可以在驱动里尝试得到应用程序传入的参数内容了。  
`WDFMEMORY                           memory;`  
`status = WdfRequestRetrieveInputMemory(Request, &memory);`  
OS已经将用户态的数据封装为IRP，FrameWork进而将IRP封装为I/O请求，我们现在要做的就是从I/O请求中得到储存数据的内存句柄，[WdfRequestRetrieveInputMemory]干的就是这个。  
得到了数据，这里我们不需要额外的处理，所以接下来就是准备发送给设备了。
PC和OSRFX2设备之间设置LEDbar的命令采用的是EP0上的Setup传输。  
`WDF_USB_CONTROL_SETUP_PACKET        controlSetupPacket;`  
`WDF_USB_CONTROL_SETUP_PACKET_INIT_VENDOR(&controlSetupPacket,`  
`                                         BmRequestHostToDevice,`  
`                                         BmRequestToDevice,`  
`                                         USBFX2LK_SET_BARGRAPH_DISPLAY,`  
`                                         0,`  
`                                         0);`  
最后的操作就是初始化其他的参数后调用最终的[WdfUsbTargetDeviceSendControlTransferSynchronously]将数据发送的USB总线上，这样设备就可以接收到了。这里我们采用的是同步发送并设置了超时。


### 2.4. Step4 - USB批量传输.




# 参考文献：  
http://www.ituring.com.cn/article/554  
http://channel9.msdn.com/Shows/Going+Deep/Doron-Holan-Kernel-Mode-Driver-Framework

[Framework Object Context Space]: http://msdn.microsoft.com/en-us/library/ff542873(v=vs.85).aspx
[device interface]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff556277(v=vs.85).aspx#wdkgloss.device_interface
[USB I/O Targets]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff544752(v=vs.85).aspx
[Using Device Interfaces]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff545432(v=vs.85).aspx
[guidgen.exe]: http://msdn.microsoft.com/en-us/library/ms241442(v=vs.80).aspx
[Framework Queue Objects]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff542922(v=vs.85).aspx
[Using I/O Control Codes]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff565406(v=vs.85).aspx
[Dispatching Methods for I/O Requests]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff540800(v=vs.85).aspx
[Handling I/O Requests in KMDF Drivers]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff543296(v=vs.85).aspx
[Framework Request Objects]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff542962(v=vs.85).aspx
[Request Handlers]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff544583(v=vs.85).aspx
[device information sets]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff541247(v=vs.85).aspx

[SetupDiGetClassDevs]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff551069(v=vs.85).aspx
[SetupDiEnumDeviceInterfaces]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff551015(v=vs.85).aspx
[SetupDiGetDeviceInterfaceDetail]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff551120(v=vs.85).aspx
[CreateFile]: http://msdn.microsoft.com/en-us/library/windows/desktop/aa363858(v=vs.85).aspx

[WdfDriverCreate]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff547175(v=vs.85).aspx
[WdfDeviceCreate]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff545926(v=vs.85).aspx
[WdfDeviceInitSetPnpPowerEventCallbacks]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff546135(v=vs.85).aspx
[WdfUsbTargetDeviceCreate]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff550077(v=vs.85).aspx
[WdfDeviceCreateDeviceInterface]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff545935(v=vs.85).aspx
[WdfIoQueueCreate]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff547401(v=vs.85).aspx
[WdfRequestRetrieveInputMemory]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff550015(v=vs.85).aspx
[WdfUsbTargetDeviceSendControlTransferSynchronously]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff550104(v=vs.85).aspx
[WdfRequestCompleteWithInformation]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff549948(v=vs.85).aspx

[DriverEntry]: http://msdn.microsoft.com/zh-cn/library/windows/hardware/ff544113(v=vs.85).aspx
[EvtDriverDeviceAdd]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff541693(v=vs.85).aspx
[EvtDevicePrepareHardware]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff540880(v=vs.85).aspx
[EvtDeviceD0Entry]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff540848(v=vs.85).aspx
[EvtIoDeviceControl]: http://msdn.microsoft.com/en-us/library/windows/hardware/ff541758(v=vs.85).aspx