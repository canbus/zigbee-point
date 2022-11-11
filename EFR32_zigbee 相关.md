# 常用API
  emberAfDebugPrintln //打印输出

# 板子配置
    EFR32MG21A020F768IM32 / EFR32MG13P732F512GM48
    Button0 PD03
    Button1 PD04
    USART0  TX/PB01
            RX/PB00
    LED0    PC01 //网络交互会闪烁
    LED1    PC00
    UART1   TX/PA06
            Rx/PA05

    PWM配置 （https://www.sekorm.com/news/74817621.html）
      isc->plugins->HAL->Bulb PWM Driver 在右边设置TIMER输出IO
                  ->Home Automation 使能Led Rgb Pwm
      add the following two defines directly in hal-config.h to solve compile issue.
        #define HAL_BULBPWM_ENABLE (1)
        #define HAL_BULBPWM_FREQUENCY                 (1000U)

# 集中式网络入网
   1. info 查看入网状态
    - 设备端panID是0x0000，nodeID是0xFFFE表示未入网，网关端panID是0xFFFF，nodeID是0xFFFE
   2. 创建集中式网络(网关端) //这步似乎不需要。直接从3开始
    - plugin network-creator start 1，创建一个ZigBee3.0集中式网络，
    - 输出EMBER_NETWORK_UP 0x0000表示建网成功，Channel是25，网关nodeID是0x0000
   2. 指定信道创建
      1. plugin network-creator form 1 4 10 26  //创建一个集中式网络,信道26，panid为0x001,发射功率为10dbm
        <uint8_t> Whether or not to form a centralized networ ...
        <uint16_t> PanID of the network to be formed
        <int8_t> Tx power of the network to be formed
        <uint8_t> channel of the network to be formed

   1. 打开允许入网(网关端)
    - plugin network-creator-security open-network
    - 打开允许入网，默认允许入网时间180秒
   2. 设备加入到ZigBee网络 (Z3Light和Z3Switch)
    - plugin network-steering start 0
    - 将设备加入到ZigBee网络中来。入网成功后，输入info命令，查看设备的panID、nodeID、Channel、Power、MAC地址以及支持的Cluster等信息
   3. 退网
    - network leave
   4. 剔除设备
    - zdo leave 0xBADF 0 0
   5. 关闭网络
    - plugin network-creator-security close-network

# 广播地址、广播、多播及路由
  一条广播消息会被网络中所有路由设备重复广播3次
  多播：同一组的设备会收到消息。而其他设备将路由这些多播消息.多播消息没有ack
  ## Many-to-One/Source Routing:Ember Stack默认使用此种路由机制
    Many-to-One路由是一种简单的机制，使得整个网络中的路由设备拥有回到中心节点的路由
      在Many-to-One路由机制下，中心节点周期性发送Many-to-One route discovery广播（协议栈默认设置为60s）,当网络中的路由设备收到这条广播之后，其拥有回到中心节点的下一跳路由，并将此跳节点信息存储在自己的路由表中.(上行路由)
    Source routing，是指中心节点将发往其它路由设备的路由机制.当每个路由设备发送单播到中心节点时，会在此之前发送一条Route Record给中心节点。中心节点收到这条Route Record，将这条路由反向并存储在中心节点的Source routing表里。这样，中心节点就可以通过查询Source routing表来获取发给目的节点的路由(下行路由)
    简而言之，只要路由设备收到Many-to-One route discovery广播，就知道回到中心节点的路由。只要中心节点的Source routing表里面有路由设备的Source routing信息，则中心节点就知道发往该路由设备的路由。Ember Stack默认使用此种路由机制。
  ## 广播地址
    0xFFFF  All devices in PAN
    0xFFFE  Reserved(nodeID为0xFFFE表示未入网)
    0xFFFD  macRxOnWhenIdle = True
    0xFFFC  All routers and coordinator

# CLI发送命令
   ## 单播命令:控制灯(让Z3Light的LED0状态反转,可以在Z3Switch/Z2Light上运行)
    - zcl on-off toggle //在消息缓冲区中填充开/关切换命令
    - send 0xE839 1 1  (0xE839是Z3Light的nodeID)
    - send (args) //单播发送命令
        <int16u> Two byte destination node id
        <int8u>  发送消息的源endpoint
        <int8u>  发送消息的目的endpoint
    - plugin device-table send {%s} 0x%02x //通过mac地址发送
    - /*plugin device-table send <deviceEui> <deviceEndpoint>*/
# CLI 绑定[ZigBee CLI绑定][]
    - 但是Z3Switch的按键控制无法控制，会跳出“Send to bindings: 0x00”,需要绑定后才能操作。
    - zdo bind 0x2834 1 1 0x0006 {000B57FFFE07B174} {000B57FFFE0BC790}
    - zdo bind 参数详情
        <int16u>  Two byte destination node id
        <int8u>  Remote device's source endpoint to bind
        <int8u>  Remote endpoint to bind
        <int16u>  Cluster on which to bind
        <string>  Remote node EUI64
        <string>  Binding's dest EUI64.  Usually the local node's EUI64


# MAC地址
  MAC地址是存储在flash中，参考：CIB manufacturing tokens for the EM35xx，地址在0x08080812可以通过ISA3烧写进去。
# Install Code
  commander.exe device info --device efr32 --serialno 20730336 
  烧录到设备:commander flash –-tokengroup znet –-tokenfile inst_001.txt --device efr32 --serialno 20730336
  验证是否烧写成功:commander tokendump –-tokengroup znet  --device efr32 --serialno 20730336

#  install code 入网
## [Day1 Zigbee技术知识基本介绍][]
## [Zigbee Boot Camp][]  ***
  * 示例程序步骤
      0. “ ZigbeeMinimal”，Zigbee最小应用程序
      1.  硬件配置：  UART0:No flow control,TX/PB01,RX/PB00
                      LED0/PC01,LED1/PC00
                      Button0/PD03,Button1/PD04
                      TIMER1:ch0/PC00
      2.  ZCL Cluster选项卡，选择HA On/Off Switch设备模板。
          Zigbee Stack选项卡，选择Router设备类型。
          Printing and CLI选项卡，检查“ Enable debug printing”是否已打开。
          Plugins选项卡，然后仔细检查以下插件是否已启用
            - Serial            对CLI必需
            - Network Steering  用来发现已启用信道中的网络
            - Update TC Link Key
            - Install code library
  * 不使用install code入网
      1. plugin network-creator start 1 创建一个ZigBee3.0集中式网络
      2. plugin network-creator-security open-network 打开允许入网，默认180秒
      3. plugin network-steering start 0   设备加入到ZigBee网络 (Z3Light和Z3Switch)
  * 使用CLI命令发送开灯命令
      1. zcl on-off on
      2. send 0 1 1
      
  * [由Light构建网络，并使用install code将Switch加入到这个网络][]
      0. install code烧录至Switch(路由器)设备
      1. 从install code中获取Link key，并将其存储到Light（作为集中式网络的Trust Center）上的Link key表
        option install-code 0 {804B50FFFE08FC33} {83 FE D3 40 7A 93 97 23 A5 C6 39 B2 69 16 D5 05 C3 B5}
        输入keys print,获得派生Link key:B0 07 D2 A6 CA 69 F7 81  B1 C3 30 49 17 7C 5F 14 
      2. 构建集中网络,Light节点上,输入以下命令
        plugin network-creator start 1
        可以用network id查询PanID
      3. 使用派生的Link key打开网络
        plugin network-creator-security open-with-key {804B50FFFE08FC33} {B007D2A6CA69F781B1C33049177C5F14}
      4. 将Switch(路由器)设备上加入网络,在Switch节点上
        plugin network-steering start 0
 
  * [在设备上使用API发送，接收和处理On-Off命令][]
    * Light设备上的命令处理
      1. “Callbacks”->“General/ OnOff Cluster”菜单下找到并启用“On”“Off”回调,然后按Generate按钮
      2. 在ZB_Light_ZC_callbacks.c中实现回调
          bool emberAfOnOffClusterOnCallback(void){
            emberAfCorePrintln("On command is received");
            halClearLed(BOARDLED1);
          }
          bool emberAfOnOffClusterOffCallback(void){
            emberAfCorePrintln("Off command is received");
            halSetLed(BOARDLED1);
          }
    * Switch设备上的命令发送
      1. 使能Button Interface插件
      2. Callbacks->Plugin-specific callBacks 启用Button0 Pressed Short和Button1 Pressed Short回调函数
      3. 在callbacks.c中实现回调
          void emberAfPluginButtonInterfaceButton0PressedShortCallback(uint16_t timePressedMs){
            emberAfFillCommandOnOffClusterOn()
            emberAfSetCommandEndpoints(1,1);//(emberAfPrimaryEndpoint(),1); //设置由哪个endpoint发送到哪个endpoint
            status=emberAfSendCommandUnicast(EMBER_OUTGOING_DIRECT, 0x0000); //单播发送到设备0x0000，然后发送到协调器
            if(status == EMBER_SUCCESS){
              emberAfCorePrintln("Command is successfully sent");
            }else{
              emberAfCorePrintln("Failed to send.Status code: 0x%x",status);
            }
          }
  * [设备事件机制使用][]
    事件由两个部分组成：
      1、  EventControl：事件的句柄。对事件的控制，都是通过这个control来执行。分别是：
      （1） 事件的激活：表示立即执行该事件。函数：
          emberEventControlSetActive(control)
      （2） 事件的挂起：表示该事件不再执行：
          emberEventControlSetInactive(control)
      （3） 事件延迟激活：表示该事件在指定的时间后执行：
          emberEventControlSetDelayMS(control, delay)
          emberEventControlSetDelayQS(control, delay)
          其中，MS的delay单位是ms，QS单位是250ms，即Quarter Second。
      （4） 事件状态的查询：查询该事件是否在激活状态：
          emberEventControlGetActive(control)
      2、  EventHandler：事件的处理函数，事件的执行体。 
        在使用事件时需要注意：
        （1）事件中不要采取阻塞式的函数，如阻塞式的delay等，这样会严重阻碍其他事件的执行。
        （2）事件的处理函数，一定要有对事件接下来的状态做处理，如果不处理，系统默认事件仍处于Active状态，会马上再度执行。





# [zigbee3讲解视频][]

## GPIO配置(hal-config.h)
  #define BSP_BUTTON0_PIN   (2U)
  #define BSP_BUTTON0_PORT  (gpioPortA)    

## TouchLink
  1. 规范中要求TouchLink过程触发需要保持在20cm范围，代码中通过调整：增加触发距离，如设置成-70dBm
      #define TOUCHLINK_WORST_RSSI -40 // dBm CC2538 
  2. 密钥
      EMBER_AF_API_UPDATE_TC_LINK_KEY
      链接密钥 #define TOUCHLINK_CERTIFICATION_LINK_KEY //CC2538
      ZStack 3.0中默认的Install code为： #define UI_INSTALL_CODE_DEFAULT

## 密钥/Key(每个节点都应该有的key)
  1. 默认的 global Trust Center link key. 中心安全.  16Bytes
  2. 默认的分布式link key. distributed security global link key. 16Bytes
  3. install code 衍生的 预配置的 link key. 
  4. touchlink 预配置的 link key (可选项)

## 学习笔记
  ### Zibgee 速率
      1. 理论速率：250kbps(2.4G,Ch11-26)/20kbps(868M,Europe/UK,Ch0)/40kbps(Americas,915M,Ch1-10)
      2. 实际速率：理论速率的四分之一或五分之一
  ### 名词解释
   1. Endpoint:每个Endpoint代表一个逻辑设备,通过实现多个Endpoint将物理设备拆分为多个逻辑设备
      0      :ZDO端口
      1-240  :App端口
      240-254: 协议使用的端口,App不可用.242用于Green Power
      255    : 广播端口
   2. ZDO:Zigbee Device Object,即Endpoint 0
   3. Zigbee设备类型:协调器/路由器/终端设备(包括休眠和非休眠设备)
   4. NCP(网络协处理器,相对于SOC概念存在)
   5. BDB的一些重要属性
      1. bdbNodeIsOnANetwork 属性指示了节点是否已加入网络
      2. bdbPrimaryChannelSet 属性指定了由应用程序定义的将优先使用的信道集
      3. bdbSecondaryChannelSet 属性指定了由应用程序定义的信道集，该信道集将在主要信道之后使用
      4. bdbScanDuration 属性指定了每个信道的 IEEE 802.15.4 扫描操作的持续时间
      5. 
  ### 属性读写(ZCL Attributes)
   1. 添加External属性 https://www.sekorm.com/news/65196344.html
   2. singleton属性(设备自身的通用属性) https://www.sekorm.com/news/21938592.html
   3. Storage属性 https://www.sekorm.com/news/22526939.html
   4. 属性的读取 https://www.sekorm.com/news/67261333.html
        zcl global read 6 0 (读一个灯设备,Cluster=6,Attribute=0)
        send 0x644f 1 1  (send [id:2] [src-endpoint:1] [dst-endpoint:1])
        返回： payload[00 00 00 10 00 ](00 00:属性, 00:成功, 10:Boolean, 00:关)
        +   读写ep1 Cluster6的属性0示例
            读ep1 Cluster6的属性0
            zcl global read 6 0 
            send 0x02 1 1
            ->payload[00 00 00 10 00 ]

            写ep1 Cluster6的属性0为1
            zcl global write 6 0 0x10 {01}
            send 0x02 1 1
            ->payload[88 00 00 ] (88:Read only)
            
            向Cluster0,属性6写入字符串"1234"
            zcl global write 0 6 0x42 "1234"
   5. 属性的写入,一般属性都是只读,如果需要改成writeable,改动如下：
      1. 修改xx_endpoint_config.h文件中的GENERATED_ATTRIBUTES表格加上ATTRIBUTE_MASK_WRITABLE
        #define GENERATED_ATTRIBUTES { \
          ....
        { 0x0000, ZCL_BOOLEAN_ATTRIBUTE_TYPE, 1, (ATTRIBUTE_MASK_WRITABLE), { (uint8_t*)0x00 } }, /* 7 on/off*/\
      2. 用以下命令写入
        zcl global write 0x06 0x00 0x10 {01}  //向ClusterID=6,attrID=0,写入boolean(0x10),值为0x01
        send 0x02 1 1                         //通过ep1发送数据到0x02的节点ep1中
   6. 属性API(UG102 P20)
      1. emberAfLocateAttributeMetadata: Retrieves the metadata for a given attribute 
      2. emberAfLocateAttributeMetadata: 取能属性
      emberAfReadAttribute/emberAfWriteAttribute
   7. 读写示例
      读取属性:
        CLI:
          zcl global read 0x06 0x00
          send 0xD9F3 1 1
        API:
          uint8_t  readLength;
          emberAfReadAttribute(1,
                              ZCL_ON_OFF_CLUSTER_ID,
                              0, 
                              CLUSTER_MASK_SERVER, 
                              &readLength, sizeof(readLength), NULL);
    写属性:
      CLI:
        zcl global write 0x06 0x0000 0x10 {01}
        send 0xD9F3 1 1
      API:
        uint8_t  writeVal = 1;
        emberAfWriteAttribute(1,
                          ZCL_ON_OFF_CLUSTER_ID,
                          0, 
                          CLUSTER_MASK_SERVER, 
                          &writeVal, ZCL_BOOLEAN_ATTRIBUTE_TYPE);


  ### CLI命令[CLI命令控制设备通讯][] / qsg106 P10   file:///C:/SiliconLabs/SimplicityStudio/v5/developer/sdks/gecko_sdk_suite/v3.2/protocol/zigbee/documentation/120-3023-000_AF_API/group__zcl-global.html
   1. zigbee 网关一些常用命令
      1. 打印设备表 plugin device-table print
      2. 设备退网 zdo leave 0x058a 0 0  usage:zdo leave [target:2（网络地址）] [removeChildren:1] [rejoin:1]
      3. 开放zigbee网络:plugin network-creator-security open-network
      4. zdo绑定:zdo bind 0x0002 1 1 6 {804B50FFFE059542} {804B50FFFE08FC2B}
          zdo bind 0x558E 1 1 6  {A4C138067532E635} {804B50FFFE08FC2B}
            请求绑定0x0002{804B50FFFE059542}到{804B50FFFE08FC2B},也就是0x0002有属性变化的时候发送到{804B50FFFE08FC2B}
          zdo bind [destid:2] [srcEp:1] [destEp:1] [cluster:2] [remoteEUI64:8] [destEUI64:8]
            destination - INT16U - 网络短地址
            source Endpoint - INT8U - Remote device's source endpoint to bind
            destEndpoint - INT8U - Remote endpoint to bind
            cluster - INT16U - Cluster on which to bind
            remoteEUI64 - IEEE_ADDRESS - Remote node EUI64
            destEUI64 - IEEE_ADDRESS - Binding's dest EUI64. Usually the local node's EUI64
      5. 根据短地址查询MAC地址:zdo ieee 0x0000
      6. 根据长地址查询短地址:zdo nwk {00 00 00 00 00 00 00} 
      7. 获取节点(0x558E)的信息:
         1. 查询邻居表/LQI:zdo mgmt-lqi 0x558E 0
         2. 查询绑定表:zdo mgmt-bind 0x558E 0
            1. zdo mgmt-bind [target:2] [startIndex:1]
                Send a ZDO MGMT-Bind (Binding Table) Request to the target device.
                target - INT16U - Target node ID
                startIndex - INT8U - Starting index into table query
         3. group及scene见 [13. zcl group 设备分组/14. zcl 设备场景scene]
         4. 获得节点的杂项信息(节点类型/供电类型/buff大小等):zdo node 0x558E
      8. 查询节点(0x558E)endpoints:
         1. 网关查询EP数量(active endpoints request):zdo active 0x558E 
         2. 节点(0x558E)回(Active endpoints Response):包含active endpoints count及active endpoints list
         3. 网关EP1的Clusters(simple descriptor request):zdo simple 0x558E 1
         4. 节点(0x558E)回(simple descriptor Response):包含指定endpoint的Input Clusters List和Ouput clusters List
      9. 获取入网设备的信息
         1. 厂商名称
          zcl global read 0x0000 0x0004
          send 0xXXXX 1 1
         2. 设备编码
          zcl global read 0x0000 0x0005
          send 0xXXXX 1 1
         3. 设备软件版本
          zcl global read 0x0000 0x4000
          send 0xXXXX 1 1
   2. info 查看入网状态
    - 设备端panID是0x0000，nodeID是0xFFFE表示未入网，网关端panID是0xFFFF，nodeID是0xFFFE
   3. 创建集中式网络(网关端) //这步似乎不需要。直接从3开始
    - plugin network-creator start 1，创建一个ZigBee3.0集中式网络，
    - 输出EMBER_NETWORK_UP 0x0000表示建网成功，Channel是25，网关nodeID是0x0000
    - plugin network-creator-security open-network是打开zigbee 3.0允许入网
    - network form是创建zigbee pro网络
   4. 打开允许入网(网关端)
    - plugin network-creator-security open-network
    - 打开允许入网，默认允许入网时间180秒
   5. 设备加入到ZigBee网络 (Z3Light和Z3Switch)
    - plugin network-steering start 0
    - 将设备加入到ZigBee网络中来。入网成功后，输入info命令，查看设备的panID、nodeID、Channel、Power、MAC地址以及支持的Cluster等信息
   6. 退网
    - network leave
   7. 单播命令:控制灯(让Z3Light的LED0状态反转,可以在Z3Switch/Z2Light上运行)
    - zcl on-off toggle //在消息缓冲区中填充开/关切换命令
    - send 0xE839 1 1  (0xE839是Z3Light的nodeID)
    - send (args) //单播发送命令
        <int16u> Two byte destination node id
        <int8u>  发送消息的源endpoint
        <int8u>  发送消息的目的endpoint
   8. 绑定[ZigBee CLI绑定][]
    - 但是Z3Switch的按键控制无法控制，会跳出“Send to bindings: 0x00”,需要绑定后才能操作。
    - zdo bind 0x2834 1 1 0x0006 {000B57FFFE07B174} {000B57FFFE0BC790}
    - zdo bind 参数详情
        <int16u>  Two byte destination node id
        <int8u>  Remote device's source endpoint to bind
        <int8u>  Remote endpoint to bind
        <int16u>  Cluster on which to bind
        <string>  Remote node EUI64
        <string>  Binding's dest EUI64.  Usually the local node's EUI64
   9.  路由端设置绑定和报告表条目(添加到绑定表)
    - ZR> option binding-table set
    - ZR> plugin reporting add
   10. zcl 读属性：
      zcl global read [cluster:2] [attributeId:2]
        cluster - INT16U – 要读取Cluster的ClusterID。
        attributeId - INT16U – 要读取属性的属性ID。
        例如，在ZigBee网关中操作：
          zcl global read 0x0006 0x0000
          send [nodeID] 1 1
   11. zcl 写属性(write 用于写本地属性)
     1. write [endpoint:1] [cluster:2] [attributeId:2] 
                [mask:1,client=0 or server=1] [datatype:1] [databytes:-1]
     2. zcl global write [cluster:2] [attributeId:2] [type:4] [data:-1]
        cluster - INT16U – 要写入Cluster的Cluster ID。
        attributeId - INT16U – 要写入属性的属性ID。
        type - INT32U – 要写入属性的数据类型
        data - OCTET_STRING – 写入的数据。
        例如，在ZigBee网关中操作：
          zcl global write 0x0006 0x0000 0x10 {01} //写OnOff(boolean)
          zcl global write 0x0006 0x4002 0x21 {1400} //写OffWaitTime = 0x14(2秒)
          zcl global write 0x0000 0xfffd 0x21 {1234}
          send [nodeID] 1 1
          
   12. zcl 属性上报
     * 属性上报绑定
       * 方式0: ZR绑定到ZC
          ZC发送:plugin find-and-bind target 1
          ZR发送:plugin find-and-bind initiator 1
        * 方式1:ZC向ZDO发送绑定请求:ZC> zdo bind 0x0002 1 1 0x0006 {804B50FFFE059542} {804B50FFFE08FC2B} 
           参数 zdo bind (args)
            <int16u>  Two byte destination node id
            <int8u>  Remote device's source endpoint to bind
            <int8u>  Remote endpoint to bind
            <int16u>  Cluster on which to bind
            <string>  Remote node EUI64
            <string>  Binding's dest EUI64.  Usually the local node's EUI64
        * 方式2:ZC向ZR发送全局ZCL发送报告命令:ZC> zcl global send-me-a-report 0x0006 0x0000 0x10 0 20 "0x0000"
                                          ZC> send 0x02 1 1
            参数：zcl global send-me-a-report (args)
              <int16u> Cluster id for requested report.
              <int16u> Attribute for requested report.
              <int8u>  Two byte zigbee type value for the requested report
              <int16u> Minimum number of seconds between reports.
              <int16u> Maximum number of seconds between reports.
              <string> Byte array. Amount of change to trigger a report.
        * 方式3:在ZR设置绑定和报告表条目在本地实现这些操作 ZC> > option binding-table set 0 0x0006 1   1 {000B57FFFE07AB24}
            参数：option binding-table set (绑定表的位置) (绑定的cluster) (Src endpoint) (Dest ednPoint) (light的EUI)
              ZR> option binding-table print //查看绑定表
              ZR> option binding-table set
              ZR> plugin reporting add
     * ZR上报属性设置
        * plugin reporting clear
        * plugin reporting add 1 6 0 1  1 60 0  //设置状态没有变化时，60秒上报一次(UG102 P45)
     * 手动上报  emberAfFillCommandGlobalServerToClientReportAttributes 

   13. zcl group 设备分组
        1. 设备加入组
          zcl groups add 0x1234 “group1”
          send 0xXXXX 1 1
          将nodeID为0xXXXX的设备端点（endpoint）1加入到组号0x1234中组名为group1
        2. 查看设备分组信息
            zcl groups view 0x1234
            send 0xXXXX 1 1
            询问NODEID为0xXXXX设备的端点1是否在groupID为0x1234中。
        3. 将设备端点从指定组中移除
            zcl groups remove 0x1111
            send 0xXXXX 1 1
        4. 将设备端点的组信息全部删除
            zcl groups rmall
            send 0xXXXX 1 1
        5. 给组成员发信息
            e.g. zcl on-off on
            send_multicast 0x1234 1
            这是向groupID 0x1234的所有设备端点1发送了一个on的命令。
   14. zcl 设备场景scene
      1. 设备端点添加场景
          zcl scenes add 0x1111 0x15 0x001e “meeting” 0x01010006
          参数分别是：[group_id] [scene_id][ transition_time] [scene_name] [SceneExtensionFieldSet]
          send 0xXXXX 1 1
          本例是以on-off cluster为例，设定在meeting场景中，目标设备的on-off属性为0x01，即on。具体拆分为：0x[attribute_value:depends_on_length] [length:8bits] [cluster_id:16bits]
          然后需要发给目标设备或组，将其加入到场景里：
          发送send [node_id] [src_endpoint] [dst_endpoint]
      2. 设备端点查看
          zcl scenes view 0x1111 0x15
          参数分别是：[group_id] [scene_id]
          send 0xXXXX 1 1
      3. 启用场景
          zcl scene recall 0x1111 0x15
          参数分别是：[group_id] [scene_id]
          send 0xXXXX 1 1或者send_multicast 0x1111 1
          参数分别是：[nodeID] [src-endpoint] [dst-endpoint] 0x1111是groupID
      4. 删除指定设备端点的指定场景
          zcl scenes remove 0x1111 0x11
          参数分别是：[group_id] [scene_id]
          send 0xXXXX 1 1
      5. 删除指定设备指定组所有场景
          zcl scenes rmall 0x1111
          参数分别是：[group_id]
          send 0xXXXX 1 1
   15. 灯控命令
      6. on/off命令
        zcl on-off on/off/toggle
        send 0xXXXX 1 1
      7. 调光命令（亮度）
        zcl level-control o-mv-to-level 0xfe 0x001e
        参数分别是：[亮度值1-254] [到达亮度所需的时间，时间单位0.1秒]
        send 0xXXXX 1 1
      8. 3色温调节
        zcl color-control movetocolortemp 0x0100 0x001e
        参数分别是：[目标色温值2700-6500][从当前色温到目标色温的时间，时间单位0.1秒]
        目标色温计算方式：1000000/0x0100≈3906K.色温参数范围[0x0099～0x0172]
        send 0xXXXX 1 1
   16. 子设备OTA升级
        1. 子设备升级限速
          e.g. plugin ota-server policy image-req-min-period 2000
          参数：每帧数据间隔时间单位毫秒
          子设备每隔2000ms请求一包OTA数据
        2. OTA升级指定设备
          e.g. plugin ota-server notify 0x607c 1 2 50 0x1002 3 0x00000002
          参数分别是：[节点ID][节点上的端点ID][指定信息负载][不用修改][生产厂商代码][Image Type ID][软件版本号]
   17. 设置绑定&信息上报(需要修改一个芯科的代码以实现组播绑定)(代码方式见:四种绑定方式(包括泰灵微))
        Note：芯科原始代码中只提供了单播绑定的CLI。
        do bind 0x1405 1 1 0x0006 {842E14FFFE9D4F88} {60A423FFFE43CFC8}
        1. 单播绑定设置
          zdo bind unicast 0x1405 1 1 0x0006 {842E14FFFE9D4F88} {60A423FFFE43CFC8}
          参数分别是[目标节点ID][信息源端点][信息目的端点][clusterID][信息源EUI][信息目的EUI]
        2. 通过绑定后配置信息上报机制
          zcl global send-me-a-report 0x0006 0 0x10 5 20 “0x0000”
          zcl global send-me-a-report 0x0006 0 0x10 5 20 {01}
          参数分别是 [clusterID][attributeID][attribute 数据类型][最小间隔时间][最大间隔时间][变化不小于该值发送report]
          send 0xXXXX 1 1
        3. 组播绑定设置
          zdo bind group 0x1405 1 0x0006 {842E14FFFE9D4F88} 0x1111
          参数分别是[目标节点ID][信息源端点] [clusterID][信息源EUI][组ID]
        4. 解除设备单播绑定
          zdo unbind unicast 0x5cdd {847127FFFE464245} 1 6 {60A423FFFE43CFD2} 1
          参数分别是：[目标节点ID][信息源EUI][信息源端点][clusterID][信息目的EUI][信息目的端点]
        5. 解除设备组播绑定
          zdo unbind group 0x5cdd {847127FFFE464245} 1 6 0x1111
          参数分别是：[目标节点ID][信息源EUI][信息源端点][clusterID][组ID]
   18. 利用HOST给NCP升级
      plugin ota-client bootload 0
      升级过程不能被打断，升级失败需要用其他方式来更新NCP
   19. 用于网关替换操作
      custom trust-center-swap-out print all//查看网关信息
      custom trust-center-swap-out save all//备份网关信息，需要实时备份
      custom trust-center-swap-out restore all//将备份的网关信息加载到新的网关硬件中
   20. 删除设备
      zdo leave 0xXXXX 0 0
      Format: zdo leave [nodeID of device you want to leave] [1=tell to leave] [1=rejoin after leave, 0=no rejoin]
      zdo leave 0xXXXX 0 1
      让指定设备退网后立即再入网
   21. 已知节点ID获取节点MAC
        zdo ieee 0xXXXX
   22. 已知节点MAC获取节点nodeID
        zdo nwk {MAC}
   23. 改变信道
        network change-channel X
        X为目标发射频道（11-26）
   24. 跟OTA文件到HOST后，使能HOST发现更新文件
        plugin ota-storage-common reload
   25. 获取子设备接收上一条信息时的RSSI&LQI
      zcl global read 0x0b05 0x011d //RSSI获取
      send 0xXXXX 1 1
      zcl global read 0x0b05 0x011c //LQI获取
      send 0xXXXX 1 1
   26. 获取指定子设备的链路质量LQI
      zdo mgmt-lqi 0xXXXX 0
   27. 通过增加device table plugin可以使用
      plugin device-table print查看所有入网设置的nodeID和端点信息
   28. 获得MAC地址  zdo ieee 0x00：获取协调器的ieee地址
   29. LQI信号质量:LQI值是一个无符号8位整数(int8u)，值越大，表示无线连接越可靠
       LQI值200映射到最低成本1、3和5，而低于200的LQI表示错误率高的链路,LQI值200表示接收完整数据包的可靠性约为80%
       RSSI和LQI关系:RSSI和LQI之间没有直接关系.
            LQI是根据当前数据包的误码率(BER)来测量连接到特定相邻无线电的链路的可靠性
            RSSI读取的只是测量给定时间内信道上的无线电能量的峰值，而不管这个无线电能量来自何处
      获取LQI方法1.在ZC上输入以下命令
        zdo mgmt-lqi 0x02 0 //读取设备的LQI值，API:emberLqiTableRequest(target, startIndex, options)
        发送zdo mgmt-lqi后,抓包:LQI Response,查看其中的Neighbor Table,可获取lqi值
      获取LQI方法2.在相应的节点查邻居表,命令如下:
        plugin stack-diagnostics neighbor-table
   30. 亮度控制 move to level
        zcl  level-control  mv-to-level 80 10  //亮度变化到80(最大255),1秒达到
        zcl level-control o-mv-to-level 250 50 //带ONoff的亮度控制
        zcl level-control o-move 1 50   //move down with on/off,10 每秒变化50
        zcl level-control o-move 0 50   //move up with on/off,10 每秒变化50 
        zcl level-control o-step 1 20 5 //变暗 20 ,5 * 0.2S之内完成
        
   31. 查看密钥: keys print 查看NWK Key/Trust Center Link Key/App link Key Table/Transient Key Table
   32. 设备识别:zcl identify id 0x10 ;send 0x02 1 1 
       1.  efr32中有判断指定厂商,同一厂商的才能控制,cli命令没有指定厂商
       2.  应用场景:当设备收到识别命令时，该设备会闪烁 LED 一段时间

  1. 主要使用的文件及目录
      <projectname>.h	                主头文件。此处列出了所有插件设置，回调设置
      <projectname>_callbacks.c	      生成的源文件。自定义回调和事件处理应在此文件中实现。
      <projectname>_endpoint_config.h	定义endpoint，属性和命令
      znet-cli.c/znet-cli.h	          CLI命令列表
      client-command-macro.h	        定义大量宏指令用于填充消息
      call-command-handler.c	        Cluster命令处理
      attribute-id.h/attribute-size.h/attribute-type.h/att-storage.h	相关属性
      af-structs.h	                  数据结构
      af-gen-event.h	                事件/处理程序对

  2. [Zigbee 新兵训练营][]
    1. [实验1:Zigbee的建网和加网][]

  + [ZigBee 3.0 学习笔记目录][]
   1. 增加、配置一个按键
   2. 增加一个LED
    void halToggleLed(HalBoardLed led);
    void halSetLed(HalBoardLed led);
    void halClearLed(HalBoardLed led);
   3. 增加一个Cluster
   4. BootLoader工程与烧录
   6. Report Attribute

  + [ZigBee 3.0-设备概念][]
   1. 角色：Coordinator、Router、EndDevice
   2. 数据发送：点播、组播、广播、绑定、Inter-PAN
   3. EndPoint，Cluster/Command/Attribute

  + zigbee 3.0 概念
    1. http://blog.chinaunix.net/uid-27875-id-5781454.html



  ### 四种绑定方式(包括泰灵微):
  1. 按键机制:(ZDP_EndDeviceBindReq)
     1. 两个节点分别通过按键机制调用ZDP_EndDeviceBindReq函数(路由器也可用)
        1. efr32:EndDeviceBindReq(void)
        2. cli:zdoEndDeviceBindRequestCommand
        3. 8258:zb_zdoEndDeviceBindReq //zb_api.h
     2. 需要协调器.绑定成功，不再需要协调器
     3. 在一定时间内(16秒)两个节点都通过按键(其他方式也可以)触发调用这个函数
     4. 这个函数的调用将会向协调器发出绑定请求,如果在16S(协议栈默认)时间内两个节点都执行了这个函数，协调器就会帮忙实现绑定
     5. 绑定表应该是存在OutCluster那边
     6. 重复上述操作会解绑定，也就是说这是一个乒乓方式。
     7. 代码
        1. 绑定表查询方式
           1. option binding-table print
           2. zdo mgmt-bind 0xF60C 0
        2. switch:cli操作
           1. custom zdoEndDeviceBindRequest 0x02 //绑定到ep02
           2. 在自定义cli命令中添加以下支持
              void myzdoEndDeviceBindRequestCommand(void)
              {
                uint8_t endpoint = (uint8_t)emberUnsignedCommandArgument(0);
                EmberStatus status = emberAfSendEndDeviceBind(endpoint);
                emberAfAppPrintln("End Device Bind %p0x%X", "Request: ", status);
              } 
        3. light:8258作server端,请求绑定onOff
            void endDeviceBindReq(void){
              u8 inNum = 1;
              u8 outNum = 0;
              u16 cluster_lst[]={0x06};
              u8 *ptr = (u8 *)&cluster_lst;
              zdo_edBindReq_t *req = (zdo_edBindReq_t*)ev_buf_allocate(6+(inNum+outNum)*2);
              if(req){
                  u8 i = 0;
                  req->binding_target_addr = g_zbInfo.nwkNib.nwkAddr;
                  req->profile_id = 0x0104;

                  u8 ieee[8];
                  flash_read(0x76000, 8, ieee);
                  u8 *p = req->src_ext_addr;
                  *p++ = ieee[6];*p++ = ieee[7];*p++ = ieee[0];*p++ = ieee[1];
                  *p++ = ieee[2];*p++ = ieee[3];*p++ = ieee[4];*p++ = ieee[5];

                  req->src_endpoint = 1;
                  req->num_in_clusters  = inNum;  //*ptr++;
                  req->num_out_clusters  = outNum; //*ptr++;
                  while (i < req->num_in_clusters){
                      //COPY_BUFFERTOU16_BE(req->in_cluster_lst[i], ptr);
                      //ptr += 2;
                      req->in_cluster_lst[i] = cluster_lst[i];
                      i++;
                  }
                  i = 0;
                  while (i < req->num_out_clusters){
                      COPY_BUFFERTOU16_BE(req->out_cluster_lst[i+req->num_in_clusters], ptr);
                      ptr += 2;
                      i++;
                  }
                  u8 seqNo;
                  zb_zdoEndDeviceBindReq(req,&seqNo,endDeviceBindReqCb); 
                  ev_buf_free((u8*)req);      
              }
              INFO("endDeviceBindReq\n");
            }
            static void endDeviceBindReqCb(void* arg){
              typedef struct{
                  u8 seqNum;
                  u8 status;
              }zdo_endDeviceBindRsp_t;

            zdo_zdpDataInd_t *p = (zdo_zdpDataInd_t *)arg;
            zdo_endDeviceBindRsp_t *rsp = (zdo_match_descriptor_resp_t *)p->zpdu;
            // ZB_LEBESWAP(((u8 *)&rsp->nwk_addr_interest), 2);
            // zbhciTx(ZBHCI_CMD_DISCOVERY_MATCH_DESC_RSP, p->length, (u8 *)rsp);

              INFO("EndDeviceBindReq:cid:%X,seqNum:%d,status:%d\n",p->clusterId,rsp->seqNum,rsp->status); 
              if(p->clusterId == 0x8020){//end device bind response
                INFO("EndDeviceBindReq:seqNum:%d,status:%X\n",rsp->seqNum,rsp->status); 
              }  
              if(rsp->status != 0)
                  INFO("EndDeviceBindReq fail\n");
          }
  2. Match方式绑定
     1. 一个节点可以通过调用afSetMatch函数允许或禁止本节点被Match(协议栈默认允许，可以手工关闭)，
     2. 另外一个节点在一定的时间内发起ZDP_MatchDescReq请求，允许被Match的节点会响应这个Req，发起的节点在接收到RSP的时候就会自动处理绑定。
     3. 特点：不需要别人帮忙，只要在网络中的节点互相之间就可以实现
     4. 注意：如果同时有多个节点(一个节点上的多个端点也一样)处于允许Match状态，那么req的这个节点可能会收到一大票满足Match条件的rsp，那么你发起req的节点要在这个处理上多下功夫了
     5. 代码:Match方式绑定
        1. match descriptor request
           1. api 函数
              1. EmberStatus matchDescriptorsRequest	(	
                  EmberNodeId 	target,
                  uint16_t 	profile,
                  uint8_t 	inCount,
                  uint8_t 	outCount,
                  uint16_t * 	inClusters,
                  uint16_t * 	outClusters,
                  EmberApsOption 	options 
                  )	
           2. zdo in-cl-list add 0x06  //设置in/out cluster list
              zdo in-cl-list add [clusterId:2] zdo out-cl-list add [clusterId:2]
           3. zdo match 0x0002 0x0104 //向短地址为0x02查询
1. bind
  3. ZDP_BindReq和ZDP_UnbindReq方式
     1. 即：应用程序通过调用这两个函数实现绑定和解绑定，这种方式很有趣。
     2. 具体说来是为了让A和B绑定到一起，还需要一个节点C。
     3. 例如：你想A控制B，那么这种方式是由C发出bind或unbind命令给A(发给谁谁就处理绑定、并负责存储绑定表)，A在接收到req的时候直接处理绑定，也就是添加绑定表项，并且这个过程B并不知道。但是A知道绑定表里面有了关于控制B的记录
     4. 并且这种方式可以实现一个节点绑定到一个Group上去。
     5. 注意：这种方式需要知道A和B的长地址
  4. 手工管理绑定表
     1. 这种方式你需要事先知道被绑定的节点信息，诸如短地址，端点号，incluster和outcluster这些信息
  5. 应用场合举例:
     1. 1、第一种方式(enddevicebind)
        例：网络中有协调器存在，另外有两个节点A和B，这两个节点具有互补特性(即A节点的incluster是B节点的outcluster)，A和B结点都有按钮可以通过程序来触发EndDeviceBind的调用。在这种情况下，你只要在规定的时间内分别按A和B上的某个按钮，绑定就会在协调器的帮助下自动完成。这适合于节点很方便操作，没有被装墙里或者无法接触的地方；绑定完成后，具有outcluster性质的B节点即可以通过绑定方式给具有incluster性质的A节点发消息，但是不能指望反过来由A发消息给B，因为A节点根本就没有关于B的信息，绑定表是存在B中的。
     2. 2、第二种方式(Match)
        例：网络中不一定有协调器存在，但是有A、B、C、D等多个节点，A性质是Outcluster，B、C、D的性质是Incluster，你可以通过按键策略来在一定时间内允许B、C、D中的任何一个开启被Match的功能，同时A发起Match请求(广播的)，那么被允许Match的节点就会在收到请求后将自己的信息返给A，A在得到rsp的时候来处理绑定，如果你还想让A绑到其它节点上，可以依次这么做。这种方式其实同第一种操作过程类似，只不过不需要协调器的参与。
     3. 3、第三种方式(Bindreq,Unbindreq)
        例：假设你有一个主控(可能是ARM板子，可能是PC……)，并且有一个Zigbee节点A通过串口或者U口等方式连在了主控上，主控可以给A发命令(什么命令你要自己定义、自己实现了)；你还有一个B节点是开关，还有一个C节点是灯。你想让在B上建立绑定表，以用来控制C。那么你可以通过主机命令A向另外的节点B发起BindReq要求，主机发给A的命令中会带着C的一些信息(主机如何有C的信息？这种场合下，主机应该了解整个网络的细节，至于如何了解。。。。，以后再说)。这样，A就可以向B发起BindReq请求，这个请求的参数中包含了必要的C的信息，B在收到请求后就会建立起关于控制C的绑定表，以后B可以通过开关控制C了，也不再需要A的参与。这种方式适合那种集中管理的网络。
     4. 4、第四种方式(手工)
        例：你还是有一个主机，主机上有Zigbee节点A能够串口或者U口通信。你想由主机来深入地管理整个网络上的绑定情况，你就可以通过主机给A发命令，告诉A给某一个节点发命令，让它去建立、删除、追加。。。。绑定表。这种方式实际上最灵活，但是需要用户(也就是你,写程序的)的工作最多，用户也需要很清晰的思路去管理绑定表，要是不小心管理错了，就杯具了
  
  ### 重新入网[rejion]
      1. 父节点:sleep enddevice 在超时时间内没有向父节点发送 date request（polling），那对应的child table entry 将失效；这个超时时间在父节点的 Zigbee PRO Stack Library 中设置（如下图）。除了超时失效外，父节点也可以主动调用EmberStatus emberRemoveChild(EmberEUI64 childEui64) 来手动移除某个子节点。当 sleep enddevice 被父节点从 child table 中移除后再次发送 data request 时，其父节点就会向其发送 leave with rejoin request。[https://www.sekorm.com/faq/116184.html]

  ### 重置
    1. 通过basic簇重置
       收到 basic 簇的 reset to factory defaults 命令时，目标设备应（SHALL）将目标设备上支持的所有簇的属性重置为其默认值。应（SHALL）保留所有其他值，如网络设置，帧计数器，分组和绑定
    2. 通过 touchlink commissioning 簇重置
       touchlink commissioning 簇提供 reset to factory new request 命令，该命令清除所有 ZigBee 持久数据（除了传出 NWK 帧计数器）（参见子条款 6.9），并执行重置以使目标处于与出厂时一样的状态
    3. 通过网络的 leave 命令重置
       清除所有 ZigBee 持久数据（除了传出 NWK 帧计数器）（参见子条款 6.9）来请求远程节点离开网络，并执行重置以使节点处于与出厂时一样的状态
    4. 通过 Mgmt_Leave_req ZDO 命令重置  
       1. 该命令通过清除所有 ZigBee 持久数据（除了传出 NWK 帧计数器）（参见子条款 6.9）来请求远程节点离开网络，并执行重置以使节点处于与出厂时一样的状态。

  ### 组及场景
    1. 需要增加ZCL Clusters中的Groups及Scenes和
        plugin中的Groups Server和Scenes Server
    2. 组建立
      添加2个灯泡进来
      zcl group add 0x03 "light" 
      send 0x02 1 1
      send 0x02 1 2

      zcl group add 0x03 "light"
      send 0x28A5 1 1		

      查看组表:plugin groups-server print
      查看场景表:plugin scenes print
    3. 测试一下组控制是否OK
      zcl on-off toggle
      send_multicast 0x03 1 	// 3组, 1为发送者的endpoint

    4. 场景建立   //0006 01 01(clusterID:6,长度1,值1:on)
      zcl scenes add [groupId:2] [sceneId:1] [transitionTime:2] [sceneName:-1] [extensionFieldSets:4]    
      zcl scenes add  0x3 0x1 1 "lightscene" 0x01010006
      send 0x02 1 1

      zcl scenes add 0x03 1 1 "lightscene" 0x01010006   //打开
      zcl scenes add  0x3 0x1 1 "lightscene" 0x01010006 0x80010008 //打开,亮度0x80
                                                        //场景需要一次性设置完所有属性,否则会异常
      send 0x28A5 1 1
      1. 场景对应的保存代码 TLSR8258_ZB_sdk\zigbee\zcl\general\zcl_scene.c中的zcl_addSceneParse()
    5. 场景执行zcl scenes recall [group_id] [scene_id] [tt](tt指多少秒后生效)
      zcl scenes recall 0x03 0x1 1
      send_multicast 0x03 1 //send_multicast [group_id] [src_endpoint]
      场景recall代码 TLSR8258_ZB_sdk\apps\sampleLight\zcl_sceneCb.c中的sampleLight_sceneRecallReqHandler()

    6. 如果想让灯的模式更丰富，比如颜色变为红色，绿色，就要在添加场景时，对不同的cluster进行设置，设置extension field sets
    目前测试只支持开、关，以及亮度值调整，已和原厂确认！以后可能会更新，或者自己想办法去实现，支持更多extension filed sets
  ### 串口使用
    #include "em_usart.h"
    #include "efr32mg21_usart.h"
    void uart_init(void)
    {
        emberSerialInit(COM_USART1, 115200, PARITY_NONE, 1);
    }
    void uart1Send(uint8_t * dataPtr, uint16_t dataLen)
    {
        while(dataLen--)//根据需要发送的数据长度判定要发出去多少个字节
        {
                while (!(USART1->STATUS & USART_STATUS_TXBL));                                                
                USART1->TXDATA = (uint32_t)(*(dataPtr++));
            }
    }
    void Uart_Recv_Data(uint8_t *buf, uint16_t length)  
    {
      uint16_t bytes = emberSerialReadAvailable(COM_USART1);
      if (bytes) {
        if (emberSerialReadData(COM_USART1, buf, length, NULL) == EMBER_SUCCESS) {
          emberSerialWriteData(COM_USART1, buf, length);
          emberSerialFlushRx(COM_USART1);
          uart1Send("rx:",3);
          uart1Send(buf,length);
        }
      }
    }
    void emberAfMainTickCallback(void)//Callbacks一栏勾选Main Tick函数
    {
        Uart_Recv_Data(array,1);
    }


  ### 色温
1:3200K
2:6000K

    4400K   4400    3868    3680    3570    3510    4950    5300    5560    5740
1:  1      0.2      0.4      0.6    0.8    1.0      0.2      0.2     0.2     0.2
2:  1      0.2      0.2      0.2   0.2    0.2      0.4      0.6    0.8    1.0


## 邻居表(neighbor-table)/路由表(ROUTE_TABEL_SIZE)[https://zhuanlan.zhihu.com/p/339646911]
  单播时,会先查邻居表,没有则查路由表,有取出路由的next hop,查next hop是否在邻居表中,在则发送,不在,发送失败,触发route request
  设备只处理邻居节点的数据,通信是需要基于邻居表的，不在邻居表内的数据包会过滤掉(https://www.sekorm.com/faq/119311.html)
  信号链路质量帧(Link Status),路由器每16秒（±2s）发一次.如果没有双向链路,发送频率提高8倍,不重试,以命令帧发送.payload包含所有的非陈旧邻居的短ID列表
  ### 邻居表:NEIGHBOR_TABLE_SIZE，默认值是16,最大26(受限于帧长127bytes)
   1. plugin启用stack diagnostics
   2. 查看邻居表:plugin stack-diagnostics neighbor-table
   3. 参数
     - 链路质量指示(LQI):是对于来自邻居所有传入的数据包的LQI进行指数加权移动平均值。邻居的传入成本(incoming cost)是使用查找表根据该值计算的。传出成本是邻居在其邻居交换消息中报告的传入成本。输出成本为0表示成本未知。如果一个条目的输出成本不为零，则称为“双向”，否则称为“单向”
     - “Age”区域存放的是自收到上一个邻居交换消息以来的时间量。新条目的age从0开始，每一个EM_NEIGHBOR_AGING_PERIOD周期增加1，默认16秒
      接收到邻居的link status数据包，age会复位成EM_MIN_NEIGHBOR_AGE(默认3),如果age大于EM_STALE_NEIGHBOR（默认为6），则该条目将被视为过时，outgoing cost将重置为0
     - 当邻居表满时，如果新邻居候选人的LQI评级比被丢弃的人高，则可以丢弃LQI最差的条目。否则，新邻居不会注册到表中
   4. 代码
       for (i = 0； i ＜ emberAfGetNeighborTableSize()； i++) { EmberStatus status = emberGetNeighbor(i, ＆n)； if ((status != EMBER_SUCCESS) || (n.shortId == EMBER_NULL_NODE_ID)) { continue； } used++； emberAfAppPrint(“%d: 0x%2X %d %d %d %d “,i,n.shortId,n.averageLqi,n.inCost,n.outCost,n.age)； }

  ### ROUTE_TABEL_SIZE
  ### 地址表:EMBER_ADDRESS_TABLE_SIZE


## [Zigbee 3.0网络优化][]
  ### 节点传输原则
    节点传播信息时会先从邻居表查询，若无则再从路由表查询，若无则发起路由请求
    为了避免不必要的高频率的节点路由请求，路由表和邻居表的大小的修改要跟你的网络规模相关，若容量够最好还是1:1甚至更高
  ### TI 400 节点网络:https://www.ti.com/lit/pdf/SWRA427D
  ### system设计优化
   1. 避免高并发情况的出现
      1. 避免在大网络中频繁广播
      2. 避免子节点同时发包(上电时发包、广播命令的应答包要加随机延时)
    2. GateWay采用many-to-one/source routing的路由方式，避免使用route request的方式建里路由 [浅谈Many-to-One/Source Routing机制][]
    3. GateWay使能High RAM concentrator,Source Route Table Size大于网络节点数[https://www.sekorm.com/news/75135548.html]
    4. Concentrator上电时需要宰现快速修复source routing的机制(需用户实现)
    5. 使能APS层重发机制,通过重发自动修复路由问题(SDK默认使用能)
    6. 如果网关同时支持其他的2.4G技术,需使能PTA(通讯包裁决机制)解决同频干扰问题
    7. 入网设备使能"Optimize scans"(SDK默认使用能),减少设备入网时信道扫描次数.并尽量增大Network Steering的时间间隔并加入随机延时,以免入网设备同时发起入网请求
  ### 快速修复源路由(source routing)方案
    当网关复位后,源路由表会被清空.此时,网关无法向子设备发送任何单播消息.需要利用以下机制快速修复源路由
    1. 当网关上电时,首先广播Many-to-one request命令
    2. 延时一段时间(如500ms)后广播一条命令(如读属性命令)到所有的子设备
    3. 子设备收到广播命令后单播发送应答命令到网关.(发送此单播消息时需要加随机延时0~3秒,以避免子设备同时发包)
  ### 利用Counters log查看设备发生各类错误的次数
     CLI command "plugin counters print"
  ### 网络参数配置文件ember-configuration-defaults.h,重点需要优化的参数
    宏定义                      默认值  诊断方法
    EMBER_PACKET_BUFFER_COUNT   75      Packet Buffer Allocate Failures
    EMBER_BROADCAST_TABLE_SIZE  15      Broadcast table full (网络中所有路由节点该参数需一致)
    EMBER_RETRY_QUEUE_SIZE      16      NWK retry overflow
    EMBER_APS_UNICAST_MESSAGE_COUNT 10  调用emberAfSendUnicast()发送单播消息返回EMBER_MAX_MESSAGE_LIMIT_REACHED(0X72)
    EMBER_SOURCE_ROUTE_TABLE_SIZE   7   需大于网络节点数

    EMBER_ROUTE_TABLE_SIZE          16  路由表stack\config\ember-configuration-defaults.h文件中 
                                        或在isc文件中,Includes->Additional Macros添加EMBER_ROUTE_TABLE_SIZE宏定义
                                        在ZigBee 3.0规范以后增加了源路由，默认使用的是源路由表，即Source route table   
## demo
  [EFR32MG裸机工程-3-按键][] 
  BootLoader工程                        https://www.sekorm.com/news/79539189.html
  创建NCP(Network Co-Processor)工程     https://www.sekorm.com/news/34372067.html
  创建Host工程                          https://www.sekorm.com/news/69088718.html
  TI demo [ZStack 3.0 SampleSwitch和SampleLight例程演示][]


## [Gecko Bootload]
    Bootloader自身对FLASH的占用是有规律的，同时也管理着整个芯片上，以及外挂载的FLASH。首先来看Bootloader本身在芯片是如何存放的。
    第一阶段引导代码，它总是占用大约2kB的空间，在Series 1系列的芯片上，这会占用一个页面（flash page）
    第二阶段主引导代码会复杂一些，不同的功能需求导致了占用空间会有多有少。在一个典型的应用中，通过不会超过14kB。所以bootloader占用的全部空间应该是在16kB以内。
    Silicon Labs也建议保留给bootloader的空间是16kB。在不同的芯片上，空间的分配又各有不同

    EFR32XG21 (含 Mighty Gecko, Blue Gecko)
    Bootloader位于FLASH程序区
    第二阶段引导地址 0x0
    应用程序首地址 0x4000

   通过simplicity commander软件将升级镜像打包为 [GBL文件格式][] ，便可以利用网关等设备对终端进行升级
## ZCL device type
    家庭自动化（HA） - 典型住宅和小型商业设施的设备。
    智能能源（SE） - 用于公用仪表读数和与家用设备的交互。
    商业楼宇自动化（CBA） - 大型商业建筑和网络设备。
    电信应用（TA） - 电信领域内的无线应用。
    医疗保健（HC） - 监控家庭或医院环境中的个人健康。
    零售 - 零售环境中的监控和信息交付。
    Zigbee Light Link(ZLL) - LED照明系统的无线控制。


## CLI 命令
#define CMD_ZCL_LOCK				   "zcl lock lock \"\""
#define CMD_ZCL_UNLOCK				   "zcl lock unlock \"\""

#define CMD_ZCL_ONOFF                  "zcl on-off %s"  /*zcl on-off on*/
#define CMD_ZCL_WINDOW_COVERING        "zcl window-covering %s"  /*zcl on-off on*/
#define CMD_RAW_WINDOW_ON              "raw 0x0102 {010000}"
#define CMD_RAW_WINDOW_OFF             "raw 0x0102 {010001}"
#define CMD_RAW_WINDOW_STOP            "raw 0x0102 {010002}"

#define CMS_TRANS_DATA                 "raw"
#define CMD_RAW_ON                     "raw 0x0006 {01%02x01}"
#define CMD_RAW_OFF                    "raw 0x0006 {01%02x00}"
#define CMD_RAW_TOG                    "raw 0x0006 {01%02x02}"
#define CMD_RAW_READ_ATTRIBUTE         "raw 0x%04x {00%02x00%02x%02x}"


#define CMD_PLUGIN_DEVICE_TABLE_SEND   "plugin device-table send {%s} 0x%02x" /*plugin device-table send <deviceEui> <deviceEndpoint>*/
#define CMD_LEVEL_CONTROL              "zcl level-control o-mv-to-level 0x%02x 0x%02x"


#define CMD_NETWORK_BROAD_PJOIN        "plugin network-creator-security open-network"  
#define CMD_NETWORK_CLOSE_NETWORK      "plugin network-creator-security close-network"
#define CMD_REMOVE_DEVICE              "zdo leave %s 1 0" /*zdo leave 0x25DC 1 0*/
#define CMD_REQUEST_DEVICES            "plugin device-table report"
#define CMD_FORM_ZB3_NETWORK           "plugin network-creator start 1"
#define CMD_SAVE_DEVICE_TABLE          "plugin device-table save"
#define CMD_REMOVE_DEVICE_TABLE        "plugin device-table remove {%s}"

#define CMD_SCENE_RECALL               "zcl scenes recall 0x%04x 0x%02x"
#define CMD_SEND_MULTICAST             "send_multicast 0x%04x 1"

#define CMD_CREAT_GROUP                "zcl groups add 0x%04x \"G%s\" "
#define CMD_SCENE_FUNC_ONOFF           "zcl scenes add 0x%04x 0x%02x 0 \"%s\" 0x%02x010006"  
#define CMD_SCENE_FUNC_LEVEL           "zcl scenes add 0x%04x 0x%02x 0 \"%s\" 0x%02x010008"
#define CMD_CLI_SEND                   "send %s 1 %d"
#define CMD_RMALL_GROUP                "zcl groups rmall"
#define CMD_MOVETO_COLORTEMP           "zcl color-control movetocolortemp 0x%04x 0x1"
#define CMD_MOVETO_HUEANDSAT           "zcl color-control movetohueandsat 0x%02x 0x%02x 0x1"

#define CMD_BIND                       "zdo bind %s 1 %d 0x%04x {%s} {%s}"

//#define CMD_ZCL_GLOBAL_SEND_ME_REPORT  "zcl global send-me-a-report 0x%04x 0x0000 0x29 01 0x0708 {3200}"
#define CMD_ZCL_GLOBAL_SEND_ME_REPORT  "zcl global send-me-a-report 0x%04x 0x%04x 0x%02x 01 0x0708 {3200}"

#define CMD_ZCL_GLOBAL_DIRECTION       "zcl global direction 0"
#define CMD_ZCL_GLOBAL_READ            "zcl global read 0x%04x 0x%04x"

#define CMD_KICK_NET_FOR_ALL           "zdo leave 0xffff 1 0"
#define CMD_PLUGIN_DEVICE_TABLE_CLEAR         "plugin device-table clear"
#define CMD_PLUGIN_COMMAND_RELAY_CLEAR        "plugin command-relay clear"
#define CMD_PLUGNIN_IAS_ZONE_CLIENT_CLEAR_ALL "plugin ias-zone-client clear-all"
#define CMD_PLUGIN_GROUPS_SERVER_CLEAR        "plugin groups-server clear"
#define CMD_PLUGIN_SCENES_CLEAR               "plugin scenes clear"

#define CMD_PLUGIN_OTA_CLIENT_BOOTLOAD0       "plugin ota-client bootload 0"

#define CMD_PLUGIN_OTA_STORAGE_COMMON_RELOAD  "plugin ota-storage-common reload"
#define CMD_PLUGIN_OTA_SERVER_NOTIFY          "plugin ota-server notify %s 0x%02x 1 1 %s %s %s"



silabs中文文档：https://github.com/dsyx/silabs.sdk.doc_zh
ug102-app-framework-guide.pdf
ug102中文：https://blog.csdn.net/lexiyao/article/details/109153280

API文档：C:\SiliconLabs\SimplicityStudio\v5\developer\sdks\gecko_sdk_suite\v3.2\protocol\zigbee\documentation\120-3023-000_AF_API
0. API是在app/framework/include/af.h中提供

emberAfFillExternalBuffer()函数会填充一个缓存，该缓存由emberAfSetExternalBuffer()函数指定
            emberAfFillExternalBuffer(uint8_t frameControl,
                                   EmberAfClusterId clusterId,
                                   uint8_t commandId,
                                   const char * format,
                                   ...)


1. 发送数据包
    1). 封装数据包,用emberAfFillExternalBuffer函数。
    2). 发送API(多播、单播、广播、绑定)
        EmberStatus emberAfSendCommandMulticast(int16u multicastId);
        EmberStatus emberAfSendCommandUnicast(EmberOutgoingMessageType  type,int16u indexOrDestination);
        EmberStatus emberAfSendCommandBroadcast(int16u destination);
        EmberStatus emberAfSendCommandUnicastToBindings(void);

    3) OnOffToggle
        emberAfFillExternalBuffer((ZCL_CLUSTER_SPECIFIC_COMMAND | ZCL_FRAME_CONTROL_CLIENT_TO_SERVER),
                            ZCL_ON_OFF_CLUSTER_ID, \
                            ZCL_TOGGLE_COMMAND_ID, \
                            "");

2. 预命令接收回调
    emberAfPreCommandReceivedCallback(EmberAfClusterCommand* cmd, boolean isInterpan)
    接收到ZCL命令但尚未被应用程序框架的命令处理代码处理之后调用。
    该命令被解析为一个有用的struct EmberAfClusterCommand，它提供了一种简单的方法来访问关于该命令的相关数据，包括它的EmberApsFrame、消息类型、源、缓冲区、长度以及该命令的任何相关标志

3. 命令回调上下文
    命令相关的回调都是从emberIncomingMessageHandler上下文中调用的

4. 全局命令回调
    设备向另一个设备发送一个全局读属性命令,使用以下命令响应
    emberAfReadAttributesResponseCallback 

5. [回调流程][]
                                 |--->[emberAfPreCommandReceivedCallback()]          |-->[emberAfPreCommandReceivedCallback()]
    emberIncomingMessageHandler()-->emberAfProcessMessage()-->emberAfCurrentCommand()__/1.emAfProcessGlobalCommand() 全局命令
                                                                                       \2.emAfProcessClusterSpecficCommand()
    1.全局命令处理                                     |->[emberAfExternalAttributeReadCallback()]         
    emAfProcessGlobal /->emberAfReadAttribute()-      |->[emberAfAllowNetworkWriteAttributeCallBack()]
    Command()         \->emberAfWriteAttributeExternal->emberAfWriteAttribute->[emberAfPreAttributeChangeCallback()]
                                                                               [emberAfPreExternalAttributeWriteCallback]
                                                                               [emberAfPostAttributeChangeCallback()]
    2.特殊Cluster命令处理
                      /--emberAfKeyEstablishmentClusterCommandParse()
    emAfprocessCluster|--emberAfScenesClusterServerCommandParse()     /--[emberAfIdentifyClusterIdentifyQueryResponseCallBack()]
    SpecficCommand    |--emberAfScensesClusterClientCommandParse   /emberAfIdentifClusterClientCommandParse()
                      \--emberAfClusterSpecificCommandParse()-----|   /--[emberAfIdentifyClusterIdentifyCallback(identifyTime)]
                                                                   \emberAfIdentifyClusterServerCommandParse()   
                                            
6. 获取endpoint和attribute信息
    api在app/framework/util/attribute-storage.h中。
    要确定一个endpoint是否包含某个属性，使用函数emberAfContainsAttribute(int8u endpoint, ClusterId, AttributeId attributeId).
    它返回一个布尔值，指示请求的属性和Cluster是否在特定endpoint上实现
    注意:读写attribute需要一个endpoint。如果不包含，编译器将返回一个警告

7. 时间处理
    emberAfGetCurrentTime()
    emberAfGetCurrentTime通过调用emberAfTimeClusterServerGetCurrentTime(实现time cluster server插件)
                             或者emberAfGetCurrentTimeCallback,返回时间

8. 事件：自定义事件/Cluster事件
    Stack的事件机制记录在位于stack/include/event.h的event.h头文件中
    创建自定义事件，打开AppBuilder配置文件中的“include”选项卡。在“Event Configuration”部分单击Add new。这将一个事件添加到将由Zigbee应用程序框架运行的事件列表中，以及AppBuilder生成的“callbacks”文件的自定义事件存根
    自定义事件由两部分组成:事件函数(在事件触发时调用)和EmberEventControl结构(用于调度事件)
    Cluster事件包括一个服务器和一个客户端“tick”回调.实际的事件表被生成到<DeviceName>_endpoint_config.h,在app/framework/util/af-event.c中使用
    1个tick大约等于1ms(1000/1024=0.98ms)
    emberAfScheduleClusterTick():定时调度  使用endpoint,cluster id关联事件
    emberAfDeactivateClusterTick():关闭事件

9. 属性管理
    1. ZCL属性配置,由以下3个文件管理
        app/framework/util/attribute-storage.c
        app/framework/utilattribute-table.c
        <appname>_endpoint_config.h
    2. ZCL属性API
        emberAfLocateAttributeMetadata:检索给定attribute的metadata
    3. “in-cluster-list”和“out-cluster-list”来代替服务器和客户端。
        “in-cluster-list”是受支持的服务器Cluster的列表，而“out-cluster-list”是受支持的客户机Cluster的列表
    4. “Identify”功能可以用来确定哪几个MAC地址对应于用户想要绑定到开关上的6个物理灯。
        使用Identify cluster，用户可以分别告诉每一盏灯“识别”自己(例如，闪烁，让它可以被看到)。
        Identify cluster定义了设备如何进入和退出Identify模式的协议

10. [ZCL命令处理][]
    emberAfProcessMessage(app/framework/util/util.c)进行命令处理,会填充EmberAfClusterCommand结构,全局指针emAfCurrentCommand
                                                              |-- process-global-message.c(处理所有全局命令，例如读取和写入属性)
    emberAfProcessMessage->emberAfPreCommandReceivedCallback--|-- process-cluster-message.c(处理所有特定于Cluster的命令)

11. 发送默认应答
    Zigbee框架不会自动为应用程序实现的命令回调发送默认响应
    Zigbee创建了一个插件来处理为他们处理的所有命令发送默认响应。插件不能处理的任何命令都会自动返回EMBER_ZCL_STATUS_UNSUP_CLUSTER_COMMAND或类似的命令
    
12. 调试打印接口
    emberAfCorePrint(…) — 打印没有回车的单行
    emberAfCorePrintln(…) — 打印带有回车的单行
    emberAfCoreFlush() — 刷新串行缓冲区
    emberAfCorePrintBuffer(buffer, len, withspace) —将给定的缓冲区打印为一系列十六进制值
    emberAfCorePrintString(buffer) — 将给定的缓冲区打印为字符串

13. [多网络支持][]
    请参阅文档AN724:在一个Zigbee芯片上设计多个网络，以获得关于此功能的详细信息
    使用emberGetCurrentNetwork和emberSetCurrentNetwork来管理当前的网络

14. [睡眠设备][]

15. [应用程序框架插件][]



16. 发送开关命令
    1.组织发送帧
    emberAfFillCommandOnOffClusterToggle();
                emberAfFillExternalBuffer((ZCL_CLUSTER_SPECIFIC_COMMAND | ZCL_FRAME_CONTROL_CLIENT_TO_SERVER), 最终调用的命令
                                        ZCL_ON_OFF_CLUSTER_ID, \
                                        ZCL_TOGGLE_COMMAND_ID, \
                                        "");
    2.发送帧
    emberAfSendCommandUnicastToBindings();
                emberAfSendCommandUnicastToBindingsWithCallback();
                        emberAfSendUnicastToBindingsWithCallback(emAfCommandApsFrame,   最终调用的命令
                                            *emAfResponseLengthPtr,
                                            emAfZclBuffer,
                                            callback);

17. 发送Identify命令
    1. emberAfSetCommandEndpoints(initiatorEndpoint, EMBER_BROADCAST_ENDPOINT);
    2. emberAfFillCommandIdentifyClusterIdentifyQuery();
                emberAfFillExternalBuffer((ZCL_CLUSTER_SPECIFIC_COMMAND | ZCL_FRAME_CONTROL_CLIENT_TO_SERVER), 
                                        ZCL_IDENTIFY_CLUSTER_ID, \
                                        ZCL_IDENTIFY_QUERY_COMMAND_ID, \
                                        "");                          
    3. emberAfSendCommandBroadcast(EMBER_SLEEPY_BROADCAST_ADDRESS);
  
18. 发送开关命令
    1. emberAfFillCommandOnOffClusterOn();
    2. emberAfSetCommandEndpoints(1,1); //设置由哪个endpoint发送到哪个endpoint .//emberAfPrimaryEndpoint();
    3. status=emberAfSendCommandUnicast(EMBER_OUTGOING_DIRECT, 0x0000); //单播发送到设备0x00(协调器)                            
    
    4. On/Off Cluster的部分cmd ID:0x00(Off)/0x01(On)/0x02(Toggle)
    5. 开关命令的交互
        switch:
            node [(>)804B50FFFE08FC33] chan [11] pwr [3]
            panID [0x185E] nodeID [0x28A5] xpan [0x(>)38AD62B16C36AAD5]
            ep 1 [endpoint enabled, device enabled] nwk [0] profile [0x0104] devId [0x0000] ver [0x01]
                out(client) cluster: 0x0006 (On/off)

            按键按下:[16:33:12.702]Command is zcl on-off ON
            收到回码:[16:33:12.830]Processing message: len=5 profile=0104 cluster=0006  
                                T00000000:RX len 5, ep 01, clus 0x0006 (On/off) FC 08 seq 06 cmd 0B payload[01 81 ]

        light:
            node [(>)804B50FFFE08FC2B] chan [11] pwr [3]
            panID [0x185E] nodeID [0x0000] xpan [0x(>)38AD62B16C36AAD5]
            ep 1 [endpoint enabled, device enabled] nwk [0] profile [0x0104] devId [0x0100] ver [0x01]
                in (server) cluster: 0x0000 (Basic)
                in (server) cluster: 0x0003 (Identify)
                in (server) cluster: 0x0004 (Groups)
                in (server) cluster: 0x0005 (Scenes)
                in (server) cluster: 0x0006 (On/off)

            收到命令:[16:33:12.760]Processing message: len=3 profile=0104 cluster=0006
                                T00000000:RX len 3, ep 01, clus 0x0006 (On/off) FC 01 seq 06 cmd 01 payload[]
                                On command is received
            回码:               T00000000:TX (resp) Ucast 0x00
                                TX buffer: [08 06 0B 01 81 ]

19. [事件的使用][] 
    1. emberEventControlSetDelayMS(定时事件)
        1. 创建自定义事件.(事件控制器–事件的结构体/事件处理程序–事件触发函数)
            AppBuilder -> Includes 将自定义事件命令ledBlinkingEventControl和回调函数ledBlinkingEventHandler分别添加到 Event Configuration窗口<img src=“pic/006.png”>
        2. 启用MainInit回调
            Zigbee_Switch_ZR.isc文件以使用AppBuilder打开它，然后在AppBuilder的“Callbacks”选项卡中启用此回调
            <img src=“pic/007.png”>
        3. 完成函数:emberAfMainInitCallback及ledBlinkingEventHandler事件处理
            回调函数emberAfMainInitCallback()应被添加到Zigbee_Switch_ZR_callbacks.c文件中并设定事件
            ```
                EmberEventControl ledBlinkingEventControl;
                void emberAfMainInitCallback(void)
                {
                    emberEventControlSetDelayMS(ledBlinkingEventControl, 5000);
                }
                void ledBlinkingEventHandler(void)
                {
                    //First thing to do inside a delay event is to disable the event till next usage
                    emberEventControlSetInactive(ledBlinkingEventControl);
                    halToggleLed(1);
                    emberEventControlSetDelayMS(ledBlinkingEventControl, 2000);//2秒后重新启用
                }
            ```
    2. emberEventControlSetActive
        1. isc->Includes->Event Configuration添加command(EmberEventControl)及callback(EventHandler)
        1. 调用emberEventControlSetActive( )将一个事件设置为Active后，需要在对应事件的EventHandler函数中，
           调用emberEventControlSetInactive( )将事件设置为Inactive，
           不然这个事件会一直运行，并不断的进入对应事件的EventHandler函数 
        2. 示例代码
            ```
            EmberEventControl ZigBeeNetworkEventControl;
            emberEventControlSetActive(ZigBeeNetworkEventControl);//系统启动自动检测网络状态
            void ZigBeeNetworkEventHandler(void) {
                emberEventControlSetInactive(ZigBeeNetworkEventControl);
                EmberNetworkStatus state = emberAfNetworkState(); //获取当前zigbee网络状态
                emberAfAppPrintln("Network State:%d",ZigBeeNetworkStatus);
                if (EMBER_JOINED_NETWORK == state) {
                    if (ZigBeeNetworkStatus != ZIGBEE_NETWORK_ALIVE)
                        ZigBeeNetworkStatus = ZIGBEE_NETWORK_UP;
                } else    // poweron/reset first entry
                {
                    if (ZIGBEE_NO_INIT == ZigBeeNetworkStatus)	//设备网络初始状态，切换到无网络状态，开始联网流程
                            {
                        ZigBeeNetworkStatus = ZIGBEE_NO_NETWORK;
                        // delay random  (delay 1s+random 1s)
                        emberEventControlSetDelayMS(ZigBeeNetworkEventControl,
                                1000 + (halCommonGetRandom() % 1000));
                        return;
                    }
                }
            }
            ```


20. 事件(按键事件)
    1. 创建自定义事件.(事件控制器–事件的结构体/事件处理程序–事件触发函数)
        AppBuilder -> Includes 将buttonPressedEventControl和buttonPressedEventHandler添加到 Event Configuration窗口
    2. 启用callbacks
        AppBuilder -> “Callbacks” ->Non-cluster related -> 启用:Hal_Button_Isr
    3. 完成函数:ButtonIsr及按键事件处理
        ```
        void emberAfHalButtonIsrCallback(int8u button,int8u state)
        {
            if((BUTTON0 == button) && (state == BUTTON_RELEASED)){
                emberEventControlSetActive(buttonPressedEventControl); //激活事件
            } 
        }
        //完成按键事件处理
        EmberEventControl buttonPressedEventControl;
        extern EmberApsFrame *emAfCommandApsFrame;
        void buttonPressedEventHandler()
        {
            EmberStatus status;
            emberEventControlSetInactive(buttonPressedEventControl);
            if(emberAfNetworkState() != EMBER_JOINED_NETWORK){
                emberAfCorePrintln("Not joined!");
            }else{
                emAfCommandApsFrame->sourceEndpoint = 1;
                //emberAfFillComamndOnOffClusterToggle();
                emberAfFillExternalBuffer(  (ZCL_CLUSTER_SPECIFIC_COMMAND | ZCL_FRAME_CONTROL_CLIENT_TO_SERVER), \
                                            ZCL_ON_OFF_CLUSTER_ID, ZCL_TOGGLE_COMMAND_ID,"");
                status = emberAfSendCommandUnicastToBindings();
                emberAfCorePrintln("%p:0x%X","send to bindings",status);
            }
        }
        ```

21. TIMER 及 PWM
    ``` void timer0() //产生10K的脉冲
    #include "em_timer.h"
    void timer0() //产生10K的脉冲
    {
    uint32_t OUT_FREQ = 1000 * 10;
    uint32_t timerFreq = 0;
    GPIO_PinModeSet(gpioPortC,1,gpioModePushPull,0);
    // Initialize the timer
    TIMER_Init_TypeDef timerInit = TIMER_INIT_DEFAULT;
    // Configure TIMER0 Compare/Capture for output compare
    TIMER_InitCC_TypeDef timerCCInit = TIMER_INITCC_DEFAULT;

    timerInit.prescale = timerPrescale1;
    timerInit.enable = false;
    timerCCInit.mode = timerCCModeCompare;
    timerCCInit.cmoa = timerOutputActionToggle;

    // configure, but do not start timer
    TIMER_Init(TIMER0, &timerInit);

    // Route Timer0 CC0 output to PC3
    GPIO->TIMERROUTE[0].ROUTEEN  = GPIO_TIMER_ROUTEEN_CC0PEN;
    GPIO->TIMERROUTE[0].CC0ROUTE = (gpioPortC << _GPIO_TIMER_CC0ROUTE_PORT_SHIFT)
                                | (1 << _GPIO_TIMER_CC0ROUTE_PIN_SHIFT);

    TIMER_InitCC(TIMER0, 0, &timerCCInit);

    // Set Top value
    // Note each overflow event constitutes 1/2 the signal period
    timerFreq = CMU_ClockFreqGet(cmuClock_TIMER0)/(timerInit.prescale + 1);
    int topValue =  timerFreq / (2*OUT_FREQ) - 1;
    TIMER_TopSet (TIMER0, topValue);

    TIMER_Enable(TIMER0, true);/* Start the timer */
    }
    ```
    ```pwm(int pwmVal)
    void pwm(int pwmVal)
    {
    uint32_t OUT_FREQ = 1000 * 10;
    GPIO_PinModeSet(gpioPortC,1,gpioModePushPull,0);
    
    TIMER_Init_TypeDef timerInit = TIMER_INIT_DEFAULT;
    // Configure TIMER0 Compare/Capture for output compare
    TIMER_InitCC_TypeDef timerCCInit = TIMER_INITCC_DEFAULT;

    timerInit.prescale = timerPrescale1;
    timerInit.enable = false;
    timerCCInit.mode = timerCCModePWM;
    
    // configure, but do not start timer
    TIMER_Init(TIMER0, &timerInit);

    // Route Timer0 CC0 output to PC3
    GPIO->TIMERROUTE[0].ROUTEEN  = GPIO_TIMER_ROUTEEN_CC0PEN;
    GPIO->TIMERROUTE[0].CC0ROUTE = (gpioPortC << _GPIO_TIMER_CC0ROUTE_PORT_SHIFT)
                                | (1 << _GPIO_TIMER_CC0ROUTE_PIN_SHIFT);

    TIMER_InitCC(TIMER0, 0, &timerCCInit);

    // Set Top value
    // Note each overflow event constitutes 1/2 the signal period
    uint32_t timerFreq = CMU_ClockFreqGet(cmuClock_TIMER0)/(timerInit.prescale + 1);
    int topValue =  timerFreq / (OUT_FREQ) - 1;
    TIMER_TopSet (TIMER0, topValue);

    TIMER_CompareSet(TIMER0, 0, (uint32_t)topValue * pwmVal / 100);
    
    TIMER_Enable(TIMER0, true);/* Start the timer */
    }
    ```


22. 自定义CLI命令(XncpLedHost_callback.c中有一个很好的示例)
    1. isc->Printing and CLI
        <img src="008.png">
    2. 添加代码
     ```
     EmberCommandEntry emberAfCustomCommands[] = {
        emberCommandEntryAction("set-led", setLedCommand, "uu", "Set the state of an LED"),
        emberCommandEntryTerminator()
    };
    static void setLedCommand(void)
    {
        uint32_t i = emberUnsignedCommandArgument(0);
        uint32_t j = emberUnsignedCommandArgument(1);
        emberAfCorePrintln("This is a test:%d",i);
    }
     ```
    3. 参数说明
        u：一字节无符号整数。
        v：两字节无符号整数
        w：四字节无符号整数
        s：一字节有符号整数
        r：两字节有符号整数
        q：四字节有符号整数
        b：字符串。可以使用引号将参数输入为ascii，例如：“ foo”。也可以使用花括号将其以十六进制形式输入，例如：{08 A1 f2}。十六进制数字必须为偶数，并且空格将被忽略。
        *：零个或多个先前的类型。如果使用，则必须是最后一个说明符。
        ？：未知数量的参数。如果使用，则必须是唯一字符。这意味着该命令解释器将不执行任何参数验证，而是直接调用该操作，并相信它将使用传入的任何参数进行处理。
        ！：可选参数定界符。

        如果命令输入具有足够的参数，使得解析器以！结尾。符号，则即使该命令不一定处理整个argumentsTypes字符串，也将被视为有效。如果除！之外还有其他参数，但仍比下一参数少！或字符串的末尾，则该命令将被视为无效。
        请注意，这使调用的CommandAction函数可以实际验证其实际获得的可选参数数量。

        示例：给定argumentsTypes字符串：uu！vv！u！vv
        以下输入类型顺序有效：
        uu，uuvv，uuvvu，uuvvuvv
        以下无效：
        u，uuv，uuvvuv，uuvvuvvv

        整数参数可以是十进制或十六进制。
        0x前缀表示十六进制整数。示例：0x3ed


# 接收消息API
    1.非基于应用框架的应用使用协议栈通用的 emberIncomingMessageHandler() 来接收和处理消息。
    2. 基于 AFV2 的应用使用各种不同的回调函数，其专门用于消息所代表的特定命令或响应，
    如 emberAfReadAttributesResponseCallback 
    或 emberAfDemandResponseLoadControlClusterLoadControlEventCallback。
# 发送消息API
    1. 发送单播
         emberSendUnicast (EmberOutgoingMessageType type,int16u indexOrDestination,
                            EmberApsFrame * apsFrame,EmberMessageBuffer message)
        emberAfSendUnicast(EmberOutgoingMessageType type,int16u indexOrDestination,
                            EmberApsFrame *apsFrame,int8u messageLength,int8u* message)

    2. 发送广播
        emberSendBroadcast (EmberNodeId destination,EmberApsFrame * apsFrame,
                            int8u radius,EmberMessageBuffer message)
        emberAfSendBroadcast (int16u destination,EmberApsFrame *apsFrame,
                            int8u messageLength,int8u* message)
    3. 发关多播:Sends a multicast message to all endpoints that share a specific multicast ID 
                and are within a specified number of hops of the sender
        emberSendMulticast (EmberApsFrame * apsFrame,int8u radius,
                            int8u nonmemberRadius,EmberMessageBuffer message)	
        emberAfSendMulticast(int16u multicastId,EmberApsFrame * apsFrame,
                            int8u messageLength,int8u* message)




link:
[事件的使用]:https://github.com/MarkDing/IoT-Developer-Boot-Camp/wiki/Zigbee-Hands-on-Using-Event-CN
[应用程序框架插件]:https://blog.csdn.net/lexiyao/article/details/109154372
[睡眠设备]:https://blog.csdn.net/lexiyao/article/details/109154326
[回调流程]:https://blog.csdn.net/lexiyao/article/details/109153371
[创建自定义CLI命令]:https://blog.csdn.net/lexiyao/article/details/109614104
[ZCL命令处理]:https://blog.csdn.net/lexiyao/article/details/109154102
[多网络支持]:https://blog.csdn.net/lexiyao/article/details/109154256

## [一些常用的zigbee设备基本工程][]
   emberznet 开发帐号
   979507902@qq.com zhm123012345
## 链接
  [ZigBee 联盟的 BDB 规范中文译本]:https://github.com/newbitstudio/ZigBeeAllianceBdbSpecification
  [ember zigbe序章:协议栈相关文档学习笔记]:https://blog.csdn.net/tainjau/article/details/90648114
  [一些常用的zigbee设备基本工程]:https://gitee.com/newbitcode/EFR32_ZigBee_Device_sls
  [浅谈Many-to-One/Source Routing机制]:http://www.newbitstudio.com/forum.php?mod=viewthread&tid=8035
  [Zigbee 3.0网络优化]:https://www.bilibili.com/video/BV1UN411Z7a2
  [CLI API online]:https://docs.silabs.com/zigbee/6.3/af_v2/group-scenes
  [CLI API]:file:///C:/SiliconLabs/SimplicityStudio/v5/developer/sdks/gecko_sdk_suite/v3.2/protocol/zigbee/documentation/120-3023-000_AF_API/group__scenes.html
  [CLI命令控制设备通讯]:https://www.sekorm.com/news/50046140.html
  [ZigBee CLI绑定]:https://www.sekorm.com/news/93291034.html
  [ZigBee 3.0-设备概念]:<https://blog.csdn.net/qq_21352095/article/details/83056040>
  [Gecko Bootload]:<https://www.cnblogs.com/newbit/p/boot3.html>
  [GBL文件格式]: https://www.cnblogs.com/newbit/p/boot4.html/ 

  [zigbee网站]:http://www.newbitstudio.com/
  [zigbee3讲解视频]:https://space.bilibili.com/99523547

  [EFR32MG裸机工程-3-按键]:https://blog.csdn.net/qq_21352095/article/details/82627389
  [ZStack 3.0 SampleSwitch和SampleLight例程演示]:https://blog.csdn.net/u011818579/article/details/88937664?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-6.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-6.no_search_link
  [ZigBee 3.0 学习笔记目录]:https://blog.csdn.net/qq_21352095/article/details/83018721
  [Zigbee 新兵训练营]:https://blog.csdn.net/weixin_48407519/category_10100301.html
  
  **  
  [实验1:Zigbee的建网和加网]:https://blog.csdn.net/weixin_48407519/article/details/106723655

  [Zigbee Boot Camp]:
  https://zhuanlan.zhihu.com/p/159370333
  [由Light构建网络，并使用install code将Switch加入到这个网络]:
  https://github.com/MarkDing/IoT-Developer-Boot-Camp/wiki/Zigbee-Hands-on-Forming-and-Joining-CN#4-%E7%83%A7%E5%BD%95%E5%B9%B6%E6%B5%8B%E8%AF%95Light%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F
[在设备上使用API发送，接收和处理On-Off命令]:
https://github.com/MarkDing/IoT-Developer-Boot-Camp/wiki/Zigbee-Hands-on-Sending-OnOff-Commands-CN
[设备事件机制使用]:https://github.com/MarkDing/IoT-Developer-Boot-Camp/wiki/Zigbee-Hands-on-Using-Event-CN
[silabs官方教程]:https://community.silabs.com/s/article/zigbee-3-0-tutorial-light-and-switch-from-scratch-step-5?language=en_US
[Day1 Zigbee技术知识基本介绍]:https://www.bilibili.com/video/BV1At4y1S7ZJ/?spm_id_from=333.788.recommend_more_video.0
[Zigbee网络中邻居表介绍]:https://zhuanlan.zhihu.com/p/339646911
[信道切换过程]:https://blog.csdn.net/luren2015/article/details/107451304	
[Host-NCP解决方案的Zigbee 3.0 Gateway]:https://blog.csdn.net/lshddd/article/details/115506219