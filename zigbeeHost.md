一个完整的网关应用分三部分：
bootloader + ncp-uart-hw + Z3GatewayHost
bootloader 选择bootloader-uart-xmodem
NCP程序选择ncp-uart-hw(硬件流控)
Host程序选择 Z3GatewayHost


# host启动
https://blog.csdn.net/lshddd/article/details/115506219
./Z3GatewayHost.exe -n 0 -f x -p com13
./bin/emqx start
\gateway-management-ui-master\nodeserver> npm run local
\gateway-management-ui-master\reactui> npm run server
浏览:http://localhost/#

gw/804B50FFFE08FC33/publishstate
gw/804B50FFFE08FC33/commands
gw/804B50FFFE08FC33/updatesettings

###
emAfOtaStorageEepromInit() //部份被屏蔽???
[ASSERT:ota-storage-eeprom-page-erase.c:188]

# OTA
OTA的开始地址必须为MCU page大小的整数倍，因为擦写的最小单位是 1 page
OTA Storage Start Offset 278528(0x44000) - 503808(0x7B000)
slot 0: start 278528  size:225280

mg21:0x50000 - 0xA0000

在HOST工程中，通过CLI：plugin ota-storage-common printImages，可以查看到Images的信息
已经入网的设备端plugin ota-client start 来进行ota升级
使用plugin ota-client info查看版本信息

# mg21 flash分布
boot: 0x00~0x4000 
flash: 33block 0x4000~0x46000
flash2: 0x48000(270336)~0x8A000(565248)
                    0x82000(532480)
                    size:262144
data:0x0b6000

=============================
0x50000-0xA0000 size:0x50000
327680-655360

0x50000-0x92000(598016) x
0x4E000(319488)-0x90000(589824)
278528(44000)-503808(7B000) x
278528(44000)-0x90000(589824)

ERROR: Next Image is too big to store (0x00041AEA > 0x0003F800)

EMBER_AF_PLUGIN_OTA_CLIENT_POLICY_FIRMWARE_VERSION 2
EMBER_AF_PLUGIN_OTA_STORAGE_SIMPLE_EEPROM_STORAGE_END 610304
============================
# EFR32MG21 zigbee 3.0 OTA 升级实验
0. https://blog.csdn.net/weixin_42160773/article/details/101284109
1. HOST：
   1. AN728中也有提到，勾选ZCL clusters->General->Over the Air Bootloading->server
   2. Printing and CLI 中勾选enable debug print以及Over the Air Bootloading Compiled in和 enable at startup
   3. Plugins 中勾选OTA Bootload Cluster Common Code、OTA Bootload Cluster Server、OTA Bootload Cluster Server Policy、OTA Bootload Cluster Storage Common、OTA POSIX Filesystem Storage Module
2. 设备端
   1. bootloader
      1. 选择bootloader-storage-internal-single，
      2. 可以勾选上      BOOTLOADER_SUPPORT_STORAGE 和 BTL_APP_SPACE_SIZE
      3. Plugins ->storage ->Common storage->start address for bootload info (524288) 
      4. Storage->Slot 0 :start(524288) size(262144)
   2. Application工程配置
      1. ZCL cluster 下面找到general，展开找到Over the Air Bootloading ,勾选client
      2. Printing and CLI 下面勾选上enable debug print ，展开Cluster debugging勾选Over the Air Bootloading
      3. HAL configuration下面的bootloader选择的是local storage
      4. Plugins下面勾选如下图所示，AN728中有很详细的写到
      5. storage的选择，这里的起始位置对应了上面bootloader的start address，这里没有选好很容易导致程序无限复位重启。



## [mg系列flash分布][]
On EFR32xG1 devices (Mighty Gecko, Flex Gecko, and Blue Gecko families), the bootloader resides in main flash.
• First stage bootloader @ 0x0
• Main bootloader @ 0x800
• Application @ 0x4000
On EFR32xG21, the main bootloader resides in main flash:
• Main bootloader @ 0x0
• Application @ 0x4000



## [mg系列切换chn,panid流程][]
条件:收到子节点unicasts fial报告,1分钟多于1条(子节点最多15分钟发送一条报告,丢包1/4以上)
emberIncomingMessageHandler->nmUtilProcessIncoming (返回true,表示不继续处理)
--> (如果在一定时间内,nmUtilProcessIncoming处理太多)->nmUtilChangeChannelRequest()(扫描通道,准备切换)
### [panid更改流程][]
panid更改参数:EMBER_PAN_ID_CONFLICT_REPORT_THRESHOLD (最大63,默认2),EZSP_CONFIG_PAN_ID_CONFLICT_REPORT_THRESHOLD 
zdo api:stack/include/zigbee-device-stack.h
panid更改:https://www.sekorm.com/news/66911675.html
### 检测 PANID 冲突
在网络上运行的任何设备都可以接收 MLME-BEACON-NOTIFY.indication 信标帧，如果信标帧的 PAN 标识符与自身的 PAN 标识符匹配，但信标有效 payload 中包含的 EPID 值不存在或不等于 nwkExtendedPANID，则应认为检测到了 PAN 标识符冲突。
// This handler is called by the
//  stack to report the number of conflict reports exceeds
//  EMBER_PAN_ID_CONFLICT_REPORT_THRESHOLD within a period of 1 minute )
EmberStatus emberPanIdConflictHandler(int8_t conflictCount)
{
  return EMBER_SUCCESS;
}
#endif

### 黑白名单:https://www.sekorm.com/news/62972683.html
PANID 过滤的具体步骤如下：

Silicon Labs的EmberZnet中的“MAC Address Filtering ”插件可以实现对PANID的管理。
这里我们的目标 PANID 是 0x1234，设置黑名单模式，命令如下：
        plugin mac-address-filtering pan-id-list add 0x1234
        plugin mac-address-filtering pan-id-list set-blacklist



## 加入指定PANID网络[https://www.sekorm.com/news/82195627.html]
在 network-steering-v2.c中找到如下定义：

void emAfPluginNetworkSteeringSetExtendedPanIdFilter(uint8_t* extendedPanId,  bool turnFilterOn)

{

  if (!extendedPanId) {

    return;

  }

  MEMCOPY(gExtendedPanIdToFilterOn,

          extendedPanId,

          COUNTOF(gExtendedPanIdToFilterOn));

  gFilterByExtendedPanId = turnFilterOn;

}



此时，我们只需要在初始化阶段设置对应的PANID即可；如果需要动态的设置，需要确保在network steering启动之前调用。需要注意，这里 extendedPanId 需要用小端模式进行定义。



## panid
//=========ncp
emAfProcessEzspCommand(EZSP_FORM_NETWORK:0x1E) ->emberFormNetwork(parameters);


//=========zc
emberAfPluginNetworkCreatorNetworkForm(bool centralizedNetwork, //仅在命令行中使用
                                                   EmberPanId panId,
                                                   int8_t radioTxPower,
                                                   uint8_t channel) 
                                                   ** ->emberFormNetwork(parameters);
scheduleScans() ->scanHandler() -> handleScanComplete() ->  tryToFormNetwork(void)  
** ->emberFormNetwork(parameters);

emberAfFormNetwork(EmberNetworkParameters *parameters) ** ->emberFormNetwork(parameters);
                                                 

typedef uint16_t EmberPanId;

// network form <channel> <power> <panid>
void networkFormCommand(void)
{
#ifdef EMBER_AF_HAS_COORDINATOR_NETWORK
  EmberStatus status;
  EmberNetworkParameters networkParams;
  initNetworkParams(&networkParams); -->networkParams->panId = (uint16_t)emberUnsignedCommandArgument(2);
  status = emberAfFormNetwork(&networkParams); --> emberFormNetwork(parameters);
  emberAfAppPrintln("%p 0x%x", "form", status);
  emberAfAppFlush();
#else
  emberAfAppPrintln("only coordinators can form");
#endif
}

void findUnusedPanIdCommand(void)
{
  EmberStatus status = emberAfFindUnusedPanIdAndForm(); -->emberAfFindUnusedPanIdAndFormCallback(void)
  emberAfCorePrintln("find unused: 0x%X", status);
}

# 配置迁移:
1. 拷贝"trust_center_swap_out_callbacks.c"和"trust_center_swap_out_callbacks.h"
2. 屏蔽"Z3GatewayHost_callbacks.c"中的"emberAfCustomCommands"
3. 修改Makefile文件,把trust_center_swap_out_callbacks.c添加到APPLICATION_FILES中
   ```
   APPLICATION_FILES= \
  ./znet-bookkeeping.c \
  ./call-command-handler.c \
  ./callback-stub.c \
  ./stack-handler-stub.c \
  ./znet-cli.c \
  ./Z3GatewayHost_2_callbacks.c \
  ./trust_center_swap_out_callbacks.c \
   ```
3. make (unix或者cygwin)
4. 生成Z3GatewayHost可执行文件
5. 运行./Z3GatewayHost.exe -n 0 -f x -p com13
6. 备份(注意此例程没有份“bind-table”和”child-table”,通过CLI命令打印出“bind-table”和”child-table”。)
   1.  custom tc-swap-out print all //查看备份的参数
   2.  custom tc-swap-out save all //将这些参数保存到backup\backup.txt文件
   3.  备份bind-table”和”child-table”
   4.  备份设备表"devices.txt"
   5.  替换硬件:断开第一个“EFR32无线通信模块”，连接上第二个“EFR32无线通信模块”，
       运行Z3GatewayHost。
       在Host端输入“custom tc-swap-out restore all”，
       第二个“EFR32无线通信模块”就恢复了之前的网络参数。
       可以通过“custom tc-swap-out print all”确认恢复后的网络参数是否正确。
       第二个“EFR32无线通信模块”网络恢复后，就可以继续和之前网络中的节点进行通信

# 参考连接:
1）《EFR32xG21搭建网关demo - 概览》：https://blog.csdn.net/xingzhibo/article/details/104535007
2）《EFR32xG21搭建网关demo - bootloader》https://blog.csdn.net/xingzhibo/article/details/107808462
3）《EFR32xG21搭建网关demo -ncp-uart-sw》https://blog.csdn.net/xingzhibo/article/details/107841658
4）《EFR32xG21搭建网关demo - Z3GatwayHost应用》 https://blog.csdn.net/xingzhibo/article/details/107907184

[mg系列flash分布]:https://blog.csdn.net/qq_21352095/article/details/110039219
[mg系列切换chn,panid流程]:https://community.silabs.com/s/article/how-does-frequency-agility-work-in-the-emberznet-pro-stack-deprecated-x?language=en_US
[panid更改流程]:https://community.silabs.com/s/article/why-is-my-network-changing-pan-ids-ember-pan-id-changed-status-when-there-are?language=en_US

[Zigbee网关替换操作指南之Host-NCP模式]:https://www.sekorm.com/news/66850141.html