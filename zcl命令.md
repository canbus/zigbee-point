# zdo命令
1. 创建网络: plugin network-creator form 1 0x1234 8 15
   1.  0x1234:pid,8是发射功率，15是信道（11~~26）
2. 切换信道:network change-channel X         
3. 开放网络: plugin network-creator-security open-network
4. 禁止设备入网: plugin network-creator-security close-network
5. 踢除节点: zdo leave 0x3152 0 0
6. 打印邻居表:plugin stack-diagnostics neighbor-table
7. 查看绑定表:option binding-table print
8. 查看路由表:plugin concentrator print-table //泰灵微 nwk_routeRecordTabEntry_t g_routeRecTab[NWK_ROUTE_RECORD_TABLE_NUM];
9. 查看密钥:keys print
10. 绑定:
    - zdo bind 0x2834 1 1 0x0006 {000B57FFFE07B174} {000B57FFFE0BC790}
    - zdo bind 参数详情
        <int16u>  Two byte destination node id
        <int8u>  Remote device's source endpoint to bind
        <int8u>  Remote endpoint to bind
        <int16u>  Cluster on which to bind
        <string>  Remote node EUI64
        <string>  Binding's dest EUI64.  Usually the local node's EUI64
11. OTA 
    - 子设备升级限速
        e.g. plugin ota-server policy image-req-min-period 2000
        参数：每帧数据间隔时间单位毫秒
        子设备每隔2000ms请求一包OTA数据
    - OTA升级指定设备
        plugin ota-server notify 0x607c 1 2 50 0x1002 3 0x00000002
        plugin ota-server notify 0x6A3D 0x01 1 12 0x1141 0x0207 0x140A3001
        参数分别是：[节点ID][节点上的端点ID][指定信息负载][不用修改][生产厂商代码][Image Type ID][软件版本号]
# 设备查询
1. 查询mac地址:zdo ieee 0xA973
2. 查询短地址:zdo nwk {680AE2FFFE6F38DF} 
3. 设备标识:zcl identify id 0x08 (8表示闪烁) ;send 0x3152 1 2
4. 查询EP数量: zdo active 0x3152
5. 查询EP2的Clusters: zdo simple 0x3152 2
6. 厂商名称获取: zcl global read 0x0000 0x0004 ; send 0x3152 1 1
7. 设备编码:zcl global read 0x0000 0x0005 ; send 0x3152 1 1
# 控制命令
1. 开关灯
    zcl on-off toggl[on/off/toggl]
    send 0x3152 1 2
2. 调光:zcl level-control mv-to-level 100 20 //2秒达到100亮度
        参数:[亮度值1-254] [到达亮度所需的时间，时间单位0.1秒]
3. 带onoff调光:zcl level-control o-mv-to-level 0xfe 0x001e
4. 亮度递加/递减:
    zcl level o-step 1 20 0 //亮度-20
    zcl level o-step 0 20 0 //亮度+ 20      
5. 色温调节:zcl color-control movetocolortemp 0x0100 0x001e
        参数：[目标色温值2700-6500][从当前色温到目标色温的时间，时间单位0.1秒]
        目标色温计算方式：1000000/0x0100≈3906K.色温参数范围[0x0099～0x0172]

6. 读属性:zcl global read [cluster:2] [attributeId:2]
   zcl global read 0x0006 0x0000  //读取6/0,clus 0x0006, id: 0x00
   send 0x3152 1 2          
   应答:payload[00 00 00 10 00 ]
                ----  |  |  \---1:on,0:off
                  |   |  \------0x10:boolean
                  |   \---------0x00:success                         
                  \-------------AttributeID:on/off
    读亮度值:zcl global read 0x0008 0x0000
    应答:payload[00 00 00 20 FF ]
                             \----亮度值
7. 写属性:zcl global write [cluster:2] [attributeId:2] [type:4] [string]
   zcl global write 0x0006 0x0000 0x10 {01} //写on/off属性

8. OTA信息:mfgId:0x1141 imageTypeId:0x0207, fw:0x14093001
   1. 读OTA版本: 
    zcl global read 0x19 0x02
    ack:[02 00 00 23 01 30 09 14 ]
                   | -----------
                   |     \---fw:0x14093001(FILE_VERSION)
                   \-unsigned 32bit Integer
   2. OTA Upgrade Status:zcl global read 0x19 0x06
    ack:[06 00 00 30 00 ]
                   \-----8 bit Enumeration
   3. Manufacture ID:zcl global read 0x19 0x07
    ack:[07 00 00 21 41 11 ]
                   | -----
                   |   \mfgId:0x1141
                   \---unsigned 16bit Integer  
   5. Image Type ID: zcl global read 0x19 0x08 
   6. plugin ota-server notify (args)
        <uint16_t>  The node ID (can be a broadcast address) to which this OTA Notify mess ...
        <uint8_t>  Target endpoint for the OTA Notify message (only really meaningful for ...
        <uint8_t>  Used to specify which parameters you want included in the OTA Notify c ...
        <uint8_t>  Corresponds to QueryJitter parameter in the OTA Upgrade cluster specif ...
        <uint16_t>  Manufacturer ID for the image being advertised (should match the mfr I ...
        <uint16_t>  Image type ID for the image being advertised (should match the image t ...
        <uint32_t>  Firmware version of the image being advertised (should match the versi ...
      plugin ota-server notify 0x9858 1 0 12 0x1002 0 30
      plugin ota-server notify 0x67A4 0x01 1 12 0x1141 0x0207 0x14093001

# ZR/ZC通用命令
1. 入网: plugin network-steering start 0
2. 退网: network leave


# raw命令 模拟OnOff命令发送流程
  格式:raw len {具体数据}   如:raw 0x0008 {013f04640000}
>zcl on-off toggle
Msg: clus 0x0006, cmd 0x02, len 3
buffer: 01 17 02
        01(0b01):command is specfic to clust
        17:transaction sequence number
        02:comamnd id[toggle]
>send 0x7e61 1 1

>raw 0x0006 {011702}
>send 0x7e61 1 1


on/off switch configuration 0x07(clusterID) 
    (Attr ID):0x00:switch type ENUM8(0x30)
    (Attr ID):0x10:switch actions ENUM8(0x30)

zcl global write 0x0007 0x0010 0x30 {01}
0x01:Off, When Arriving At State 2 From State 1 / On, When Arriving At State 1 From State 2
0x00:On, When Arriving At State 2 From State 1 / Off, When Arriving At State 1 From State 2

# 未整理
广播使用的nodeID:
0xFFFF广播给所有的在网设备
0xFFFD广播给非睡眠的在网设备
0xFFFC广播给在网的所有路由设备

创建ZigBee集中式网络
e.g. plugin network-creator form 1 0x1234 8 15
0x1234是ZigBee网络的PANID可随机生成，8是发射功率，15是信道（11~~26）


设置绑定&信息上报(需要修改一个芯科的代码以实现组播绑定)
Note：芯科原始代码中只提供了单播绑定的CLI。
zdo bind 0x1405 1 1 0x0006 {842E14FFFE9D4F88} {60A423FFFE43CFC8}

9.1单播绑定设置
zdo bind unicast 0x1405 1 1 0x0006 {842E14FFFE9D4F88} {60A423FFFE43CFC8}
参数分别是[目标节点ID][信息源端点][信息目的端点][clusterID][信息源EUI][信息目的EUI]

9.2通过绑定后配置信息上报机制
zcl global send-me-a-report 0x0006 0 0x10 5 20 “0x0000”
zcl global send-me-a-report 0x0006 0 0x10 5 20 {01}
参数分别是 [clusterID][attributeID][attribute 数据类型][最小间隔时间][最大间隔时间][变化不小于该值发送report]
send 0xXXXX 1 1

9.3组播绑定设置
zdo bind group 0x1405 1 0x0006 {842E14FFFE9D4F88} 0x1111
参数分别是[目标节点ID][信息源端点] [clusterID][信息源EUI][组ID]

9.4解除设备单播绑定
zdo unbind unicast 0x5cdd {847127FFFE464245} 1 6 {60A423FFFE43CFD2} 1
参数分别是：[目标节点ID][信息源EUI][信息源端点][clusterID][信息目的EUI][信息目的端点]

9.5解除设备组播绑定
zdo unbind group 0x5cdd {847127FFFE464245} 1 6 0x1111
参数分别是：[目标节点ID][信息源EUI][信息源端点][clusterID][组ID]

利用HOST给NCP升级
plugin ota-client bootload 0
升级过程不能被打断，升级失败需要用其他方式来更新NCP。

用于网关替换操作
custom trust-center-swap-out print all//查看网关信息
custom trust-center-swap-out save all//备份网关信息，需要实时备份
custom trust-center-swap-out restore all//将备份的网关信息加载到新的网关硬件中
网关替换操作代码参考链接，来源于芯科官方论坛

删除设备
zdo leave 0xXXXX 0 0
Format: zdo leave [nodeID of device you want to leave] [1=tell to leave] [1=rejoin after leave, 0=no rejoin]

zdo leave 0xXXXX 0 1
让指定设备退网后立即再入网。

设备类型支持的cluster获取
zdo simple 0xXXXX 1
参数分别是：[节点ID][节点上的端点ID]

获取入网设备的信息
14.1厂商名称
zcl global read 0x0000 0x0004
send 0xXXXX 1 1

14.2设备编码
zcl global read 0x0000 0x0005
send 0xXXXX 1 1

14.3设备软件版本
zcl global read 0x0000 0x4000
send 0xXXXX 1 1

已知节点ID获取节点MAC
zdo ieee 0xXXXX

已知节点MAC获取节点nodeID
zdo nwk {MAC}

切换ZigBee网络的工作频率
network change-channel X
X为目标发射频道（11-26）

跟OTA文件到HOST后，使能HOST发现更新文件
plugin ota-storage-common reload

获取子设备接收上一条信息时的RSSI&LQI
zcl global read 0x0b05 0x011d //RSSI获取
send 0xXXXX 1 1

zcl global read 0x0b05 0x011c //LQI获取
send 0xXXXX 1 1

获取指定子设备的链路质量LQI
zdo mgmt-lqi 0xXXXX 0

raw命令使用
e.g. 读取OTA 软件版本号
raw 0x0019 {undefined08 00 00 02 00}
0x0019 clusterID 08 server-to-client 00 命令序列号 00 read attribute command ID 02 00 实际是0x0002 attribute ID
send 0xXXXX 1 1

通过增加device table plugin可以使用
plugin device-table print查看所有入网设置的nodeID和端点信息

