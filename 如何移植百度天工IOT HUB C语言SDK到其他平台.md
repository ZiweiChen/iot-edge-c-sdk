### 本文档主要介绍如何移植C公共库到其他平台（目前还没有支持的平台）。
### 目前百度天工的IoT Hub的C语言SDK和BOS的C语言SDK都是试用了这个C公共库。
*** 
### 本文档不会特定介绍哪一个具体的平台移植方案，是一个通用的移植方案。
*** 
### 参考
*** 
### 规范

•	tickcounter adapter specification

•	agenttime adapter specification

•	threadapi and sleep adapter specification

•	platform adapter specification

•	tlsio specification

•	xio specification

•	lock adapter specification

 ### 头文件列表

•	tickcounter.h

•	threadapi.h

•	xio.h

•	tlsio.h

•	socketio.h

•	lock.h

•	platform.h

***
### 目录结构
### 整体概括
tickcounter 适配器
agenttime适配器
sleep适配器
platform 适配器
tlsio adapter 适配器
socketio 适配器概括(可选)
tlsio适配器实现
threadapi and lock适配器(可选)

***
### 概括
目前采用C99的标准编写的C共享库主要考虑改标准可以很方便的移植到大部分的平台。但是还有部分模块需要依赖系统相关的资源来实现对应的功能。因此，这个C共享库提供PAL（platform abstraction layer）模块来让这个库可以适配到特定平台。 下面就是这个模块的整天架构:
![](https://github.com/baidu/iot-edge-sdk-samples/blob/master/STM32/B-L475E-IOT01/_htmresc/hub1.png)


### 必须适配的几个模块如下：
1. tickcounter的实现：这个接口提供一个可以获取以毫秒为单位时钟计数器。精度不一定必须是毫秒级别，但是这个接口的返回值必须是以毫秒为单位。
2. agenttime的实现： 这个接口提供c运行时的函数，包括获取time，difftime等等。这个主要由于不同平台对具体实现的处理方式不一样。
3. sleep函数的实现提供一个平台无关的sleep逻辑
4. platform接口提供执行全局初始化和反初始化的逻辑
5. tlsio接口提供标准的基于TLS之上的通讯方式。 IoT Hub不允许不安全接入方式。

另外，有两个可选的模块，他们是threadapi和lock，这两个模块允许SDK通过特定的线程来进行数据通讯。另外一个需要适配的模块是socketio适配器，这个需要配合tlsio不同实现方式。

目前改SDK已经包含了一个标准适配实现，你可以在SDK的特定目录下找到他们。如果其中的一些适配器满足你的设备需求的话，可以直接引用这些文件到你的项目。

### tickcounter适配器
Tickcounter的行为定义在tickcounter的spec
你可以通过拷贝一份tickcounter.c文件，通过部分修改来适配你的设备
如果内存大小是一个问题的话，tickcounter spec提供了如何优化方案

### agenttime适配器
agentime适配器的逻辑在agenttime适配器spec里面做了详细介绍以及如何提供一个平台无关的time函数

大部分的平台和os可以使用标准agentime.c放到你的编译环境，这个适配器只是简单的调用c的time, difftime, ctime 等等。

如果这个文件在你的平台无法工作的，你可以拷贝一份，做适当的修改就可以了

百度IoT SDK只需要get_time和get_difftime这两个函数。get_gmtime, get_mktime和get_ctime已经被deprecated了，可以直接忽略不管。

### sleep适配器
这个sleep适配器只是提供一个函数，提供一个平台无关的线程sleep函数。 不想其他适配器，这个适配器没有自己的头文件。他是将这个函数定义在threadapi.h，这个头文件还包含其他的一些可选的适配函数，这个函数的实现通常定义在threadapi.c文件。

除了ThreadAPI_Sleep是SDK必须的函数，threadapi.h里面其他的函数都是可选的。

sleep适配器的spec可以在threadapi and sleeep适配spec里面找到

### platform适配器
platform适配器主要执行一次性平台的初始化和反初始化同时也提供SDK的TLSIO相关的支持

Platform适配器spec给出详细的介绍关于如何编译platform适配器

你可以通过拷贝一份platform.c文件，通过修改来适配自己的需求

### threadapi和lock适配器
编译SDK必须包含threadapi和lock这个两个适配器，但是他们内部逻辑实现是可选的。他们的spec文档详细介绍如果threading相关逻辑不需要的话，其他空函数应该怎么做。

这个模块允许SDK和IoT Hub交互通过指定的线程。特定的线程执行特定任务有很多额外的开销由于需要为每个线程分为独立的stack，这样对于某些资源受限的设备来说，就比较困难了。

试用特定线程执行tlsio相关逻辑的好处就是可以尝试重新连接IoT Hub当网络不可用是，同时还有其他线程可以处理其他逻辑，这样程序的响应会比较及时。

以后版本的SDK会移除潜在的blocking的行为，不过现在任然需要指定线程来处理消息，提高系统的响应，这样就需要实现ThreadAPI和Lock适配器

### 下面是threadapi和lock适配器的spec
threadapi and sleep adapter specification
lock adapter specification

这个spec介绍如何去创建null lock和threadapi的适配器当线程不需要的时候。

如果需要创建自己的threadapi和lock适配器，可以拷贝windows的适配文件，通过适当修改来满足你的需求：
threadapi.c
lock.c

### tlsio适配器介绍
tlsio适配器提供SDK可以通过标准的安全的TLS通讯方式让device和IoT Hub交互数据


Tlsio适配器通过xio接口暴露功能让SDK来调用，这个接口提供一个输入为bits，返回也为bits的接口，具体定义可以访问：https://github.com/Azure/azure-c-shared-utility/blob/master/inc/azure_c_shared_utility/xio.h

通过调用函数xio_create来创建tlsio适配器实例，在创建tlsio实例的时候，你必须配置参数const void *io_create_parameteres，这个参数是TLSIO_CONFIG的一种类型。

tlsio有两种模式
tlsio支持的模式包括两种：直接的，串联的。在直接模式，tlsio适配器直接创建自己的TCP socket，试用TCP直接和远程的服务器进行TLS通讯。在串联模式下，tlsio适配器不拥有自己的TCP socket，不直接和远程服务器通讯，但是它任然处理TLS的所有逻辑，包括加密，解密，协商，只不过通过另外的方式和远程服务器进行通讯。

直接模式的好处是消耗资源会少很多，比较适合MCU，比如Arduino和ESP32等等。串联模式提供更大的灵活度，但是资源消耗也会多很多，所以主要主流的OS，比如windows，linux和Mac等等

### socketio适配简介
串联的tlsio适配器必须调用xio的适配器，这个适配器可以包含一个tcp socket。在baidu的IoT SDK里面，xio适配器是通过socketio管理tcp socket。

下面的接个图介绍tls的数据流模型：
![](https://github.com/baidu/iot-edge-sdk-samples/blob/master/STM32/B-L475E-IOT01/_htmresc/hub2.png)


### tlsio适配器实现
开发tlsio适配器功能相对来说还是很复杂的，需要很有经验的开发人员才能完成，需要对协议有很深的理解。


tlsio适配器相关的逻辑都是在C共享库实现的，不是baidu IoT SDK的一部分。你可以参考下面的资料去了解如何开发。

有两个直接模式tlsio适配器实现：
tlsio_openssl_conpact for ESP32
https://github.com/Azure/azure-iot-pal-esp32/blob/master/pal/src/tlsio_openssl_compact.c
tlsio_arduino for Arduino
https://github.com/Azure/azure-iot-pal-arduino/blob/master/pal/src/tlsio_arduino.c

tlsio_openssl_compact for ESP32是一个好的例子，如果需要适配的话，可以拷贝一份，基于这个修改来满足自己的需求。
tlsio_openssl_conpact for ESP32提供类两个文件，这两个文件是和具体平台无关的
socket_async.c
https://github.com/Azure/azure-c-shared-utility/blob/master/pal/socket_async.c
dns_async.c
https://github.com/Azure/azure-c-shared-utility/blob/master/pal/dns_async.c

大部分的用户都可以直接使用这两个文件而不需要修改，对于特殊情况只需要修改socket_async_os.h就可以了

支持设备支持仓库

### 推荐参考
https://github.com/Azure/azure-iot-pal-esp32，来创建一个自己的仓库。


