# silicon lab
    https://www.silabs.com/support/training.ap-language_mandarin.soft-development-environments_simplicity-studio.soft-software-development-kits_zigbee-sdk
    https://docs.silabs.com/simplicity-studio-5-users-guide/latest/ss-5-users-guide-overview/

    1.zigbee基础：https://github.com/dsyx/silabs.sdk.doc_zh/blob/master/docs/UG103.2%20Zigbee%20Fundamentals.md

# 待学习
    https://www.zhihu.com/people/TorchIoTBootCamp
    Zigbee 新兵训练营入口: https://zhuanlan.zhihu.com/p/159370333  

# 问题
    zigbee 集中式网络/分布式网络区别

# [EFR32芯科zigbee学习文档资源总结][] 
  [Zigbee 新兵训练营][]

# Endpoint: application object. like logical device.
            ep0:        Zigbee Device Object(ZDO),used for config/admin
            ep1-239:    for user applications
            ep240-254 :  for special functions
            ep255:      Used for Broadcastging for all endpoints
# Cluster: communication model
            Client/Server model
            Defined in Zigbee Cluster Library(ZCL)
            Commands
            Attributes

# 相关参数
    5 max_depth 网络最大深度
    6 max_children 路由器或者协调器子节点最大个数
    20 max_router  路由器或者协调器处理的具有路由能力子节点的最大个数
    一个节点除了64位的 IEEE 地址，16位的网络地址，每个节点还提供了8位的应用层入口地址（端点：EndPoint)
    对应于用户应用对象。EP0为ZDO接口,1~240给用户使用。255广播,241~254保留
    
    每一个应用都对应一个配置文件(profile).配置文件内容包括：设备ID(deviceID),事务ID(ClusterID),属性ID(Attribute ID),及AF使用何种服务类型等信息。
    一个配置文件中允许最多2^16个设备,2^8个事务，每个事务支持最多2^16个属性

# 网络参数在ember-configuration-defaults.h
    可以通过counters log监控参数配置
    参数宏定义                      默认值      诊断方法
    EMBER_PACKET_BUFFER_COUNT       75      Packet Buffer Allocate Failures
    EMBER_BROADCAST_TABLE_SIZE      15      Broadcast table full
    EMBER_RETRY_QUEUE_SIZE          16      NWK retry overflow
    EMBER_APS_UNICAST_MESSAGE_COUNT 10      调用emberAfSendUnicast()发送单播消息返回
                                            EMBER_MAX_MESSAGE_LIMIT_REACHED(0x72)
    EMBER_SOURCE_ROUTE_TABLE_SIZE   7       需大于网络节点数

# ZCL概念
    ZCL是Zigbee 1.1（Zigbee2006）协议版本中增加的一个重要的部分。在Zigbee中，一个簇群就是一个容器
    容器中以命令结构体包含了一个或多个属于某个应用剖面的属性/消息
    相同的设备（比如开关）拥有相同的定义和功能。属性是设备的变量或特性，能够设置或获得

    ZCL提供了一种机制，利用这种机制设备能够将变化异步地报告给属性（attribute），比如当空气变热时自动控温器服务器就将室温改变报告给他的客户端，这个过程不需要客户端发起请求

    ZCL采用客户端/服务器模块的模式，一般储存簇属性的作为服务器，影响或操作属性的作为客户端

    cluster ID是每个簇的标志

# 一个ZCL应用至少需要创建4个模块：
  1. zcl_<appname>.h 应用的定义,应用的终端也定义在此
  2. zcl_<appname>_data.c 数据定义和声明，包含以下内容
    应用支持的所有簇属性；
    属性表中每个属性包含一个zclAttrRec_t类型的入口；
    分别包含应用指定的输入和输出cluster ID的输入cluster ID表和输出cluster ID表，这些表将和简单描述表一起使用；
    SimpleDescriptionFormat_t[AF.h]类型的简单描述表。
  3. zcl_<appname>.c
    endPointDesc_t[AF.h]类型的应用终端表声明；
    创建用以处理来自ZCL簇的命令的命令回调函数，这些函数和命令回调表一起使用；
    ZCL功能域（functional domains）的应用命令回调表声明。通用功能域的表类型为：zclGeneral_AppCallbacks_t[zcl_general.h]；
    为应用任务创建应用初始化函数void zcl<AppName>_Init( byte task_id )；此函数应该注册：
        1、相应功能域的命令回调表，zclGeneral_RegisterCmdCallbacks()[zcl_general.c]用来注册通用簇命令回调；
        2、用zcl_registerAttrList()[zcl.c]注册应用属性列表；
        3、用afRegister()[AF.h]注册应用终端；
        4、用RegisterForKeys()[OnBoard.c]注册所有处理按键事件的应用任务；
        5、用zclHA_Init()[zcl_ha.c]注册HA剖面层的应用简单描述。

    创建用于接收和处理消息以及应用任务队列中关键事件的函数uint16 zcl<AppName>_event_loop( uint8 task_id, uint16 events )。
  4. OSAL_<AppName>.c
    此模块应该包含void osalAddTasks( void )函数，此函数包含添加到任务列表中的应用所需的任务和应用任务本身。使用osalTaskAdd()[OSAL_Tasks.c]来完成任务的添加。在此展示一个最小的任务列表，他们的添加顺序是一个简单ZCL应用所需的：
    1. HAL
    2. MAC
    3. Network
    4. APS
    5. ZD Application
    6. ZCL
    7. ZCL Application



# Zigbee传入的消息处理流程图
   <img src="pic/001.png" width=80% height=80%>

# 回调函数
   AppBuilder生成的回调函数在callback-stub.c,启用后,应该从该文件中删除,并添加到<ProjectName>_callbacks.c中

# 与Cluster无关的回调
   由 callbacks.xml 文档中描述的回调组成
   在用户表示希望接收到 Zigbee 应用程序框架相关功能消息的位置，这些回调被人为插入到 Zigbee 应用程序框架代码中。所有全局命令都属于这一类
   任何全局命令回调返回 TRUE，则表明该命令已被应用程序处理，并且没有进一步的命令处理应该发生。如果回调返回 FALSE，则 Zigbee 应用程序框架继续正常处理命令。
   全局命令的处理流程:
   <img src="pic/002.png" width=80% height=80%>

# Cluster相关的命令处理回调
   Cluster相关的命令处理回调由 Zigbee 应用程序框架生成，以允许接收预先解析的无线命令。
   ZCL 命令和Cluster相关的命令处理回调之间存在一对一的关系
   回调返回TRUE，则框架认为已处理(默认响应已发送?)。返回 FALSE，发送“unsupported cluster command”响应。
   Cluster相关的命令处理流程图：<img src="pic/003.png" width=80% height=80%>

# 事件(Event):自定义事件和cluster事件
   自定义事件由用户创建，并且可以在应用程序中随意使用。
   cluster事件由Zigbee应用框架插件中的cluster实现方式决定。
   自定义事件包括两部分：事件处理函数（在事件触发时调用）和EmberEventControl结构体（用于设定事件）。
   Zigbee应用程序框架和AppBuilder提供了一个图形化界面，用于创建自定义事件并将其添加到应用程序中。
   
# 属性管理(Attribute Management)：
    在Zigbee应用程序框架中，由AppBuilder产生的两个源文件(attribute-storage.c，attributetable.c)和一个头文件(_endpoint_config.h)管理属性的存储。
    1. 外部属性(External Attributes)
        在AppBuilder中属性旁边的选择框中选"E"，系统每次需要的时候都是从外部硬件中读取属性的值。
    2. 持久性内存存储器(Persistent Memory Storage)
        在AppBuilder中属性旁边的选择框中选"F"，属性保存在非易伯性存储器(NVM3)中
    3. 单独存储(Singleton)
        所有EndPoint公用同一个属性地址
    4. 属性绑定(Attribute Bounding)
        如果ZCL中属性有最大值和最小值限制，可以设置属性绑定。
    5. 属性报告(Attribute Reporting)
    6. ZCL字符串属性
        第一位是字符串长度，没有结束字符。例如“05 68 65 6C 6C 6F” 表示ZCL字符串“hello.”

# 插件(Plugin)：
    插件是通过Zigbee应用程序框架的回调函数接口在Zigbee应用程序框架内实现的一些功能部件。
    Zigbee 应用程序框架附带许多Zigbee Cluster的默认处理代码
    常用的插件：
        Zigbee PRO Stack Library 具有路由支持的核心协议栈，由路由和协调器使用
        Zigbee PRO Leaf Library 不支持路由的核心协议栈，由终端设备使用
        End Device Support 一个支持终端设备的插件
        Network Creator 创建网络，由协调器使用
        Network Creator Security 协调器的安全设置，例如为新设备配置Link key
        Network Steering 扫描可加入的网络并加入
        Idle/Sleep 由睡眠终端设备使用。空闲时设备将进入EM2。
        EM4 插件可帮助睡眠终端设备进入EM4
        Simple Main 项目的主要入口
        Simulate EEPROM Version 1 Library 模拟EEPROM版本1的库，用于存储非易失性数据
        Simulate EEPROM Version 2 Library 模拟EEPROM版本2的库，用于存储非易失性数据
        NVM3 Library NVM3库，用于存储非易失性数据

# 命令行接口(CLI) 
    https://docs.silabs.com/zigbee/latest/zigbee-af-api/cli
    https://docs.silabs.com/gecko-platform/latest/service/cli/overview
    源码：znet-cli.c和{*}-cli.c
    Zigbee应用程序框架中包括一个命令行接口 (CLI)
    命令行接口(CLI)使用一种开源的通信方式实现了许多常见的命令和Cluster相关的命令。 
    例如，经常使用的，如网络创建，加网和属性读写等操作都可以用CLI轻松实现。
    CLI命令:
        1.plugin counters print 查看设备发生各类错误的次数
        2.plugin network-creator start 1 创建一个ZigBee3.0集中式网络
        3.plugin network-creator-security open-network 打开允许入网，默认180秒
        4.plugin network-steering start 0   设备加入到ZigBee网络 (Z3Light和Z3Switch)
        5.info
        6.option binding-table print 查看绑定表
          plugin address-table print 地址表
          plugin stack-diagnostics neighbor-table 邻居表
          plugin stack-diagnostics child-table  子节点表
        7.keys print 查看NWK Key/Trust Center Link Key/App link Key Table/Transient Key Table
        8.raw命令使用
            e.g. 读取OTA 软件版本号
            raw 0x0019 {undefined08 00 00 02 00}
            0x0019 clusterID 08 server-to-client 00 命令序列号 00 read attribute command ID 02 00 实际是0x0002 attribute ID
            send 0xXXXX 1 1

    
    CLI发送命令
        * 直接发送命令
            1.填充ZCL命令: zcl on-off toggle
            2.发送命令：send [nodeID:2] [src-ep:1] [dst-ep:1]
        * 绑定
            3.在灯的ep1上允许绑定：plugin find-and-bind target 1
            4.在灯的ep1上启动find and bind流程：plugin find-and-bind initiator 1
        * 输入参数绑定(在ZC上输入)
            zdo bind 0x28A5     1       1     0x0006  {804B50FFFE08FC33}    {804B50FFFE08FC2B}
                     nodeID sourceEP destEP   cluster   remote EUI64        Binding's dest EUI64

    用户自定义的CLI
        1.在AppBuilder的Includes标签中添加一个宏定义"MBER_AF_ENABLE_CUSTOM_COMMANDS"
        2.在应用程序中定义一个名称为emberAfCustomCommands，类型为EmberCommandEntry的数组
        3.下面是“form” and “join”命令行接口的示意
            EmberCommandEntry networkCommands[] = {
            { “form”, formCommand, “uvsh” },
            { “join”, joinCommand, “uvsh” },
            };
        1. 一般来说，命令行接口由三部分组成，字符串命令，函数，字符串参数

# 休眠的终端设备(Sleepy end devices)
    休眠终端设备是大部分时间都处于断电状态的 Zigbee设备，Zigbee 应用程序框架包含对休眠终端设备的支持。 休眠终端设备只有在需要执行某些特定操作(例如 GPIO中断或轮询其父节点以查看网络上是否有任何消息)时才为处理器通电

    轮询(polling)
        休眠终端设备不直接从网络上的其他设备接收数据。他们必须轮询其父级以获取数据
        父级充当休眠终端设备的代理，在孩子(休眠终端设备)休眠时保持清醒和缓冲消息。
        APS重试超时（7.8 秒）和终端设备轮询超时（由父节点定义，默认值为 5 分钟）。
        这两个超时对应休眠终端设备上的两个轮询间隔：SHORT_POLL 和 LONG_POLL 。
        分别称为"nap duration"和"hibernation duration"
        
# Zigbee版本简介：
    <img src="pic/004.png" width=80% height=80%>
    
# Zigee入网简易过程
    1.设备重启后会首先调用emberNetworkInit()。
        如果设备之前加入过网络,emberNetworkInit()会返回EMBER_SUCCESS，否则返回EMBER_NOT_JOINED。
        即使返回EMBER_SUCCESS，也不能保证终端设备在完成此操作后立即处于EMBER_JOINED_NETWORK状态;它只表示复位之前的设置能够恢复，现在该节点必须尝试找到合适的父设备(child table中具有可用空间的router或coordinator)。
    2.emberNetworkInit()成功,接着调用StackStatusHandler回调：
        2.1 原父节点仍然在网络中且child table还存该终端设备，则为EMBER_NETWORK_UP;
            emberNetworkState()返回EMBER_JOINED_NETWORK，这意味着可以发送和接收消息
        2.2 原父节点的child table已清除此终端设备或者原父节点已经不在网络，则为EMBER_MOVE_FAILED。
            emberNetworkState()返回EMBER_JOINED_NETWORK_NO_PARENT，需要重新入网
    3.节点进入EMBER_JOINED_NETWORK_NO_PARENT：可以通过2种方式解决
        3.1 调用emberPollForData()尝试轮询,这会导致EMBER_ERR_FATAL，然后果协议栈会启动自动rejoin网络查找新的父节点 
        3.2 调用emberFindAndRejoinNetwork(TRUE,0),强制在具有相同NWK加密密钥的同一信道上重新加入同一网络。
        3.3 在Rejoin中,StackStatusHandler回调会EMBER_NETWORK_DOWN,入网后EMBER_NETWORK_UP

# [绑定][],[绑定1][]
    1.概念:绑定机制允许一个应用服务在不知道目标地址的情况下向对方（应用服务）发送数据包
        时间由参数APS_DEFAULT_MAXBINDING_TIME决定，默认为16秒
    2.绑定信息保存在Zigbee协调器中?,所有只有协调器才能接收绑定请求
    3.方式1:调用ZDP_EndDeviceBindReq,该操作是乒乓操作,需要协调器帮忙,绑定后不需要协调器
        节点B(outcluster)和C(incluster)分别通过按键调用ZDP_EndDeviceBindReq,向协调器A发出绑定请求
        如果在16S内两个节点都执行了此函数，协调器就会协助实现绑定。
        绑定表放在OutCluster那边，即绑定表存放在输出控制命令的那边。
        绑定后只能是outcluster节点B给incluster节点C发控制命令，因为只有B中保存C的信息(绑定表)
    4.方式2:Match方式
        节点B通过调用afSetMatch函数允许或禁止本节点被Match(默认允许,可以关闭)
        节点C在一定的时间内发起ZDP_MatchDescReq请求,B节点会响应这个Req
        节点B在接收到RSP的时候就会自动处理绑定。
        特点:不需要别人帮忙。注意：如果同时有多个节点(一个节点上的多个端点也一样)处于允许Match,req个节点会收到一大票满足Match条件的rsp，那么你发起req的节点要在这个处理上多下功夫
    5.方式3:ZDP_BindReq和ZDP_UnbindReq方式
        应用程序通过调用这两个函数实现绑定和解绑定,可以实现一个节点绑定到一个Group上
        A要控制B:C节点发出bind命令给A.A收到req的时直接处理绑定(添加绑定表项)。
        注意：需要知道A和B的长地址。
    6.方式4:手工管理绑定
        通过调用bindAddEntry
        注意：需要知道被绑定的节点信息：短地址、端点号,incluster,outcluster

# General ZCL Frame帧结构(on/off命令为例)
   <img src="pic/005.png">
    bits:8          0/16        8       8   Variable
    Frame control   制造商代码  帧序列号 CID  Frame payload(ZCL payload)

    Frame Control:8位
        bit0-1: 帧类型(4种:Beacon,Data,Ack,MAC Command), 0b1 表示是特定或者本在的cluster的"on/off cluster"
        bit2  : 制造商指定, on/off 中设为false
        bit3  ：方向位, 0b0 表示switch到light

# EmberStatus,错误码，状态
    0x66  EMBER_DELIVERY_FAILED  没有可用的替代路径，消息无法传递



# 涂鸦zigbee接入标准,产品为基于标准的 Zigbee 3.0 协议
   ##   Profile Id	0x0104
        Device Id	0x0051
   ##   Endpoint
        1           用于应用数据交互时的 endpoint


# zigbee 码流
    TX buffer: [08 00 0B 02 81 ]       

[绑定]:https://blog.csdn.net/chenbang110/article/details/53868940
[绑定1]:https://blog.csdn.net/shy19910509/article/details/85060042?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-0.no_search_link&spm=1001.2101.3001.4242

[Zigbee 新兵训练营]:https://blog.csdn.net/weixin_48407519/category_10100301.html
[EFR32芯科zigbee学习文档资源总结]:https://zc-iot-wireless.blog.csdn.net/article/details/109998541?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-5.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-5.no_search_link