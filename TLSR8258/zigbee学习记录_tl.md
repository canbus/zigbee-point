## Flash分布 Telink_Zigbee_Overview_CN.pdf P23
* 512K Flash (524288)
  * 多地址模式
    * 固件:0x00和0x40000(OTAImage)
    * 0x76000:4096 MAC_Addr      前8个Bytes MAC,出厂预写. 见"## IEEE地址存储"
    * 0x77000:4096 F_Cfg_info    出厂配置信息(频偏,ADC校准等)
    * 0x78000~0x7A000:8K U_Cfg_info    install code等
    * 0x34000~0x40000:0xC000 (49152)   NV_1
    * 0x7A000~0x80000:0x6000 (24576)   NV_2 网络信息存储区,恢复网络参数
## dongle:A4:C1:38:06:75:32:E6:35 0x0000 ch13
   开发板:076000:  7a 49 ff 38 c1 a4 7b 61 
## 协调器参数
    zigbee每增加一个节点就会增加ram占用，单个设备连接100个节点，运行效率和空间都不太能满足，建议你增加路由设备扩展网络数量，TL_ZB_CHILD_TABLE_NUM可以增加父节点连接子节点的数量。
    将2个表都改大即可，TL_ZB_NEIGHBOR_TABLE_NUM 包含了自己子节点的数量。

## OTA升级机制 https://blog.csdn.net/libin55/article/details/118763975
修改编译版本:STACK_BUILD
固件大小应不大于208K

地址标识 0x20000地址划分

MCU上电后会默认从0地址启动，首先去读0x08地址的数据信息，值为0x4B时会从0地址开始搬移代码到RAM，指令取指也从0地址开始
如果值不为0x4B，MCU会去读取0x20008地址的值，如果读取到的值为0x4B，则代码搬移和指令取指会从0x20000地址
0x40000地址划分
同上，标识地址为0x00008与0x40008，必须通过接口配置bls_ota_set_fwSize_and_fwBootAddr
//设置最大的firmware size 为 252K
bls_ota_set_fwSize_and_fwBootAddr(200, 0x40000);//200k, 4byte对齐


## 打印输出
  * 打印使能(app_cfg.h)     #define UART_PRINTF_MODE    1
  * 打印口配置(board_xx.h)   #define DEBUG_INFO_TX_PIN    GPIO_PC4
                            #define BAUDRATE    1000000//1M
## 版本管理 version_cfg.h
  * 制造商代码  #define MANUFACTURER_CODE_TELINK    0x1141 //(Telink)
  * 固件类型    #define IMAGE_TYPE   ((CHIP_TYPE << 8) |MAGE_TYPE_SWITCH)  
  * 文件版本    #define FILE_VERSION    ((APP_RELEASE << 24) | (APP_BUILD << 16) | (STACK_RELEASE << 8) | STACK_BUILD)

## 库文件
8系列可以在POST步奏的tl_check_fw.sh文件的最后使用tc32-elf-ar.exe -r app.a app.o 来命令封装对应的库

tc32-elf-ar.exe -r E:/telink_zigbee_ble_concurrent_sdk/platform/lib/app_ble.a E:/telink_zigbee_ble_concurrent_sdk/build/tlsr_tc32/concurrent_sampleLight_8258/apps/sampleLight/app_ble.o
以上命令添加到E:\telink_zigbee_ble_concurrent_sdk\tools\tl_check_fw.sh

lib存放:E:\telink_zigbee_ble_concurrent_sdk\platform\lib

## API
  * drv_platform_init(void)       初始化,完成芯片/系统clock/gpio/rf/timer等模块的初始化
  * 射频
    * ZB_RADIO_INIT()               RF初始化
    * ZB_RADIO_TRX_SWITCH(mode,chn) 模式切换
      * mode: RF_MODE_TX/RF_MODE_RX/RF_MODE_OFF
      * chn:  11~26 
    * ZB_RADIO_TX_POWER_SET(level)  发射功率. 参考RF_PowerIndexTypeDef(platform/chip/rf_drv.h)
    * ZB_RADIO_TX_START(txbuf)      数据发送
      * txBuf:待发送数据的内存地址
      * 数据格式:dma length(4Bytes:palyload长度+1) + len(1Byte:payload长度+2) + payload
    * ZB_RADIO_RX_BUF_SET(addr)     设置射频接收buffer
    * ZB_RADIO_RSSI_GET()           RSSI获取
    * ZB_RADIO_RSSI_TO_LQI(mode,rssi,lqi) RSSI转Lqi
      * mode:仅对8269有效
    * rf_rx_irq_handler(void)       接收中断
    * rf_tx_irq_handler(void)       发送中断
  * GPIO
    * gpio_init() 
    * drv_gpio_func_set(u32 pin)  设置IO具体功能
      * pin:参见GPIO_PinTypeDef定义
    * drv_gpio_read(u32 pin)                  读取gpio电平
    * drv_gpio_write(GPIO_PD3,0)      设置gpio的高低电平
    * drv_gpio_output_en(u32 pin,bool enable) 输出使能
    * drv_gpio_input_en(u32 pin,bool enable)  输入使能
    * drv_gpio_up_down_resistor(TEST_GPIO, PM_PIN_PULLDOWN_100K); 下拉
    * 中断操作
      1. 同时支持3路外部GPIO中断,模式:GPIO_IRQ_MODE,GPIO_IRQ_RISC0_MODE和GPIO_IRQ_RISC1_MODE
      2. 注册中断服务函数:drv_gpio_irq_conf(mode,pin,polarity,callback)
      3. 使能中断管教:
         1. drv_gpio_irq_en(pin)
         2. drv_gpio_irq_risc0_en(pin)
         3. drv_gpio_irq_risc1_en(pin)
      ```
        void gpio_irq_callback(void)
        {
            drv_gpio_write(LED1, 1); 
            INFO("irq\n");  
        }
        void irq_init()
        {
            drv_gpio_func_set(GPIO_PD6);
            drv_gpio_output_en(GPIO_PD6, 0); 		//disable output
            drv_gpio_up_down_resistor(GPIO_PD6, PM_PIN_PULLUP_10K);
            drv_gpio_input_en(GPIO_PD6, 1);		// enable input

            drv_gpio_irq_config(GPIO_IRQ_MODE,GPIO_PD6,FALLING_EDGE,gpio_irq_callback);
            drv_gpio_irq_en(GPIO_PD6);
        }
      ```
    
  * UART
    * 管教设置: drv_uart_pin_set(txPin,rxPin) 使用uart前必须选择相应的IO作为收发管脚
    * 初始化: drv_uart_init(bps,rxBuf,rxLen,recvCB)
    * 发送:   drv_uart_tx_start(data,len)
    * 接收中断函数:drv_uart_rx_irq_handler(void)
    * 发送中断函数:drv_uart_tx_irq_handler(void)
    * 异常处理函数:drv_uart_exceptionProcess(void)
  * ADC
    1. 初始化: drv_adc_init(void)
    2. 配置:drv_adc_mode_pin_set(drv_adc_mode_t mode,GPIO_PinTypeDef pin) 
    3. 获取采样值 drv_get_adc_data(void)
  * PWM
    gpio_set_func(GPIO_PC1, AS_PWM1_N); 	
    drv_pwm_n_invert(1); 
    pwm_set_clk(CLOCK_SYS_CLOCK_HZ, 8000000);
    pwm_set_cycle_and_duty(1, 8000, 0);
    drv_pwm_start(1);

    gpio_set_func(GPIO_PC0, 30); //AS_PWM4 	
    drv_pwm_n_invert(4); 	
    pwm_set_clk(CLOCK_SYS_CLOCK_HZ, 8000000);
    pwm_set_cycle_and_duty(4, 8000, 4000);
    drv_pwm_start(4);
    pwmSetDuty(4, 10); 

  * TIMER
    * drv_hwTmr_init()
    * drv_hwTmr_set()
    * drv_hwTmr_cancel()
    * 回调函数注意事项:
      * 不可以在自身的任务中取消
      * 返回小于0,表示任务只执行一次.执行完后自动注销
      * 返回等于0,表示任务执行后,仍然使用注册的时间参数来启动定时任务
      * 返回大于0,表示任务执行后,使用该返回值作为新的时间参数来启动定时任务
    * 中断
      * drv_timer_irq0_handler(void)
      * drv_timer_irq1_handler(void)
      * drv_timer_irq3_handler(void)
  * Watch
    * #define MODULE_WATCHDOG_ENABLE  1 //看门狗启用,
    * drv_wd_clear();//喂狗
    * drv_wd_setInterVal(ms)
    * drv_wd_start
    * drv_wd_clear
  * SystemTick
    * 8285最大定时:268秒
    * clock_timer 获取当前tick
  * 电压检测
    * 需要使用支持ADC功能的IO口
    * 使用DRV_ADC_VBAT_MODE,IO口需要悬空
    * 使用DRV_BASE_VBAT_MODE,,IO口需要连接电压测试点
    * voltage_detect_init
    * voltage_detect
  * 睡眠和唤醒  AN_19052901 P60
  * 存储管理    AN_19052901 P61
  * NV管理
  * 任务管理
    * 单次任务:只执行一次,没有优先级,要避免一次压入过多任务 
            TL_SCHEDULE_TASK(tl_zb_callback_t func, void *arg)
    * 常驻任务:启动后,在主循环中一直执行
            注册:ev_on_poll(ev_poll_e e, ev_poll_callback_t cb)
            启动:ev_enable_poll(ev_poll_e e, ev_poll_callback_t cb)
            暂停:ev_disable_poll(ev_poll_e e, ev_poll_callback_t cb) 
    * 软件定时:
            注册:TL_ZB_TIMER_SCHEDULE(ev_timer_callback_t func, void *arg, u32 t_ms)
            取消:TL_ZB_TIMER_CANCEL(ev_timer_event_t **evt)
    * 使用实例 AN_19052901 P65
      * SW1按下后,启动或取消一个10秒定时任务,执行广播on/off Toggle命令
  * 数据发送
    * 使用ZCL命令接口
      * read/write/report等基础命令
      * cluster命令
    * 使用AF Data发送接口:af_dataSend()
        * u8 srcEp
        * epInfo_t *pDstEpInfo,目标端口信息
        * u16 clusterId,cluster ID
        * u16 payloadLen
        * u8 *payload
        * u8 *apsCnt,当前消息的aps counter
  * 数据接收: zcl_rx_handler(void *pData)
    * 使用af_endpointRegister()注册消息接收处理函数
    * typedef struct apsdeDataInd_s{
          aps_data_ind_t indInfo;//源地址,目的地址等
          u16 asduLen;
          u8 asdu;//payload
        } 
  * 系统异常处理:sys_exceptHandlerRegister()
    * 调用sys_exceptHandlerRegister注册异常回掉函数.异常时,通过变量T_evtExcept[1]可以定位发生了何种异常
  * install code
    * 网关导入目标设备:zb_pre_install_code_store(addrExt_t ieee,u8 *pInstallCode)
    * 待入网设备从Flash加载使用:zb_pre_install_code_load(app_linkkey_info_t *appLinkKey)
  
## 常用API
  ### AF
  1. af_endpointRegister()  注册端点描述符
  2. af_dataSend()  使用AF层发送消息
  3. bdb_defaultReportingCfg()  绑定后生效
  4. zb_setPollRate(u32 newRate) 设置data request发送速率,终端设备使用
  5. zb_rejoinReq() 终端设备丢失父节点后调用
  6. zb_factoryReset() 恢复出厂设备.恢复前会广播离网通知 
  7. zb_mgmtPermitJoinReq() 允许入网命令
  8. zb_mgmtLeaveReq()  离网命令
  9. zb_mgmtLqiReq()    获取目标设备的关联邻居表信息
  10. zb_mgmtBindReq()  获取目标设备的绑定表信息
  11. zb_mgmtNwkUpdateReq() 更新网络参数或者请求网络环境信息
  12. zb_zdoBindUnbindReq() 发送绑定或者解绑命令
  13. zb_zdoNwkAddrReq()    请求目标短地址
  14. zb_zdoIeeeAddrReq()   request to target device for IEEE address
  15. zb_zdoSimpleDescReq() Send simple descriptor request
  16. zb_zdoNodeDescReq()   Send node descriptor request
  17. zb_isDeviceJoinedNwk() 检测节点是否已经入网

## 应用开发
  1. 需要修改的参数
     1. 运行模式  #define BOOT_LOADER_MODE 0         //comm_cfg.h
     2. 芯片型号  #define CHIP_TYPE  TLSR_8258_512K  //version_cfg.h
     3. 目标板    #define BOARD  BOARD_8258_EVK      //app_cfg.h
     4. 修改CLUSTER数量 #define	ZCL_CLUSTER_NUM_MAX	//stack_cfg.h
  2. UART打印
     * 打印使能(app_cfg.h)        #define UART_PRINTF_MODE    1
     * 打印口配置(board_xx.h)     #define DEBUG_INFO_TX_PIN    GPIO_PC4 //print
     * 波特率设置(drv_putchar.h)  #define BAUDRATE    512000//1000000//8258最大1M
  3. USB打印
     1. #define USB_PRINT_MODE 1
     2. #define HW_USB_CFG()  do{ usb_set_pin_en();}while(0)
     3. 当使用ZBHCI_USB_CDC或者ZBHCI_USB_HID功能时,USB打印功能失效
  4. zdo注册协议栈回调:void zb_zdoCbRegister(zdo_appIndCb_t *cb)  
     可注册的函数
         1. zdpStartDevCnfCb
         2. zdpResetCnfCb
         3. zdpDevAnnounceIndCb
         4. zdpLeaveIndCb
         5. zdpLeaveCnfCb
         6. zdpNwkUpdateIndCb
         7. zdpPermitJoinIndCb
         8. zdoNlmeSyncCnfCb
         9. zdoTcJoinIndCb 
  5. 端口描述符回调:af_endpointRegister() 
     1. u8 ep
     2. af_simple_descriptor_t *simple_desc
     3. af_endpoint_cb_t rx_cb 接受消息回调,通常是zcl_rx_handler()
     4. af_dataCnf_cb_t cnf_cb 发送消息的确认回调
  6. cluster处理函数:zcl_register()  
     1. u8 endpoint
     2. u8 clusterNum 支持的cluster数量
     3. zcl_specClusterInfo_t *info, clusters和attributes注册表
        1. u16 clusterId 
        2. u16 manuCode 厂商代码
        3. u16 attrNum 该cluster支持的属性数量
        4. const zclAttrInfo *attrTbl 属性表
        5. cluster_registerFunc_t func ,cluster注册函数指针
        6. cluster_forAppCb_t appCb ,cluster消息回调函数
  7.  网络参数配置:/tl_zigbee_sdk/zigbee/common/zb_config.c
     4. APS_BINDING_TABLE_NUM,绑定表个数
     5. APS_GROUP_TABLE_NUM,分组表个数
     6. TL_ZB_NWK_ADDR_MAP_NUM,地址映射表个数(影响到网络容量,512最大127个节点,1M 300个节点),关联表?
     7. TL_ZB_NEIGHBOR_TABLE_NUM,邻居表个数,路由默认26个(最大26,受限于mac层帧长127字节)
        1. g_zb_neighborTbl
     8. ROUTING_TABLE_NUM,路由表个数
     9. NWK_ROUTE_RECORD_TABLE_NUM,路由记录表个数
     10. NWK_BRC_TRANSTBL_NUM,广播表个数
     11. NWK_ENDDEV_TIMEOUT_DEFAULT,终端设备保活超时时间
     12. ZB_MAC_RX_ON_WHEN_IDLE,终端设备空闲时RF接收机的状态
     13. ZB_NWK_LINK_STATUS_PEROID_DEFAULT ,link status命令周期时间
  8.  pwm
      drv_pwm_init();
      R_LIGHT_PWM_SET();
              do{	\
                    gpio_set_func(GPIO_PD4, AS_PWM2_N); 	\
                    drv_pwm_n_invert(2); 	\
                }while(0)
      pwmInit(2, 80);
      drv_pwm_start(2);
      drv_pwm_stop(2);
    
        //直接操作PC0,PC1
        gpio_set_func(GPIO_PC0, 30); //AS_PWM4 	
        drv_pwm_n_invert(4); 	
        pwm_set_clk(CLOCK_SYS_CLOCK_HZ, 8000000);
        pwm_set_cycle_and_duty(4, 8000, 4000);
        drv_pwm_start(4);
        //pwmSetDuty(4, 10); 

        gpio_set_func(GPIO_PC1, AS_PWM1_N); 	
        drv_pwm_n_invert(1); 	
        pwm_set_clk(CLOCK_SYS_CLOCK_HZ, 8000000);
        pwm_set_cycle_and_duty(1, 8000, 4000);
        drv_pwm_start(1);

  11. 软件定时器
      s32 light_blink_TimerEvtCb(void *arg)
      {// @return  0: timer continue on; -1: timer will be canceled
      }
      static ev_timer_event_t *levelTimerEvt = NULL;
      levelTimerEvt = TL_ZB_TIMER_SCHEDULE(light_blink_TimerEvtCb, NULL, 100);//push timer task to task list
      TL_ZB_TIMER_CANCEL(&levelTimerEvt);
  12. 属性自动上报
      //设置60秒上报一次.有更改立即上报
      u8 reportableChange = 0x00;
      bdb_defaultReportingCfg(SAMPLE_LIGHT_ENDPOINT, HA_PROFILE_ID, ZCL_CLUSTER_GEN_ON_OFF, ZCL_ATTRID_ONOFF,
    						0x0000, 60, (u8 *)&reportableChange);
      在协调器中:zdo bind 0x892C 1 1 6 {A4C138067532E635} {804B50FFFE08FC2B}
  13. 串口
      u8 uartRxBuf[20]={0};
      void uartRxIrqHandler(void){
        //the format of the uart rx data: length(4 Bytes) + payload
        drv_uart_tx_start((u8 *)&uartRxBuf+4,uartRxBuf[0]);
      }
      void uartInit()
      {
        UART_PIN_CFG();//uart_gpio_set(UART_TX_PIN, UART_RX_PIN);
        drv_uart_init(115200, uartRxBuf, sizeof(uartRxBuf)/sizeof(u8), uartRxIrqHandler);
        drv_enable_irq();
        drv_uart_tx_start("hello",5);
        MYTRACE("uartInit");
      }
  14. 出厂设置
      1. 关键变量 factoryRst_powerCnt
      2. 快速上下电10次,每次2秒内,恢复出厂
  15. 属性上报(代码):
      u8 srcEp = 1;
			epInfo_t dstEpInfo = {0};
			#if 0
				dstEpInfo.dstAddrMode = APS_DSTADDR_EP_NOTPRESETNT; //绑定表发送
				dstEpInfo.profileId = 0x104;
			#else 
				dstEpInfo.dstAddrMode = APS_SHORT_DSTADDR_WITHEP; //指定短地址发送
				dstEpInfo.dstAddr.shortAddr = 0x0002;
				dstEpInfo.dstEp = 1;
				dstEpInfo.profileId = 0x0104;
			#endif 
			u8 clusterID = 6;
			u8 attriID = 0;
			#if 0
				u8 attriType = 0x10; //boolean
				u8 data = 1;    //上报boolean型
			#else 
				u8 attriType = 0x42; //字符串
				u8 data[] = {0x04,'1','2','3','4'}; //上报字符串
			#endif
			zcl_sendReportCmd(srcEp, &dstEpInfo,  TRUE, ZCL_FRAME_SERVER_CLIENT_DIR,
								clusterID, attriID, attriType, &data);
  16. 发送私有数据(代码)
      void sendData(u8 srcEp,u8 dstEp,u16 dstShortID,u8* txBuf,u8 len)
      {
          epInfo_t dstEpInfo = {0};

        dstEpInfo.dstEp = dstEp;
        dstEpInfo.profileId = 0x0104;
        dstEpInfo.dstAddrMode = APS_SHORT_DSTADDR_WITHEP;
        dstEpInfo.dstAddr.shortAddr = dstShortID;

        u8 dataLen = len;
        u8 *pBuf = (u8 *)ev_buf_allocate(dataLen);
        if(pBuf){
          u8 *pData = pBuf;
          for(u8 i = 0; i < dataLen ; i++){
            *pData++ = *txBuf++;
          }

              u8 apsCnt = 0;
              af_dataSend(srcEp, &dstEpInfo, ZCL_CLUSTER_TELINK_SDK_TEST_REQ, dataLen, pBuf, &apsCnt);
              ev_buf_free(pBuf);
          }
      }
  17. 自定义命令
      1. msg_level=[0,1,2,3]  //无调试信息/简单调试信息/加Tag的调试信息
      2. getIeee  //获取mac地址
      3. getRssi
      4. mfTest,1,2 //100ms发送一报,测试20次
      5. zclOn,1    //打开ep1的灯
      6. zclOff,1   //关闭ep1的灯
      7. reboot
      8. factoryReset
  18. 启动时初始化灯的亮灭
      1. zcl_levelAttr_t g_zcl_levelAttrs =
        {
            .curLevel				= 0xFE,
            .remainingTime			= 0,
            .startUpCurrentLevel 	= ZCL_START_UP_CURRENT_LEVEL_TO_PREVIOUS,//上电LED灯状态灭掉
        };
      2. 启动时初始化函数zcl_level_startUpCurrentLevel(u8 endpoint);




## sampleLight主要流程
  * sampleLight_onOffCb 开关灯回调,在sampleLightEpCfg.c g_sampleLightClusterList表中做关连
  * sampleLight_levelCb 亮度调节回调

## 开关灯控制流程
  sampleLight_onOffCb->sampleLight_onoff
  1.开关
    zcl on-off toggle ;send 0x5A44  2 1
  2.查询on/off状态
    zcl global read 0x06 0x00 ;send 0x5A44  2 1

## 测试命令
zcl color-control movecolor 0x0010 0x0020  rateX:16 steps pecond,rateY:32 steps per second
zcl color-control movetocolor 0 0xFF 1 	 color x:0 color y:255  1/10 second

## 亮度控制
zcl level-control mv-to-level 10 20 //20/10秒之内亮度渐变到10%
send 0x5A44  2 1

## 颜色控制流程(zigbee学习系列之Color Control:https://zhuanlan.zhihu.com/p/121091685)
标准白点:x=0.33,y=0.33
3000K,x=0.440、y=0.403,6000K,x=0.323,y=0.325
zcl color-control movetocolor 0x704A 0x672A 2  //x=0.44*65536=28836 y=0.403*65536=26410  
send 0x5A44  2 1
sampleLight_colorCtrlCb->sampleLight_moveToColorProcess->sampleLight_updateColorMode

hue,sat控制    hue(0°~360°)  s(0~100%)   (默认v=100)
zcl color-control movetohueandsat 0 0 1 //0°,0%,100%->#FFFFFF 0,100%,100%->#FF0000  120,100%,100%->#00FF00 240,100%,100%->#0000FF
sampleLight_colorCtrlCb->sampleLight_moveToHueProcess->sampleLight_updateColorMode
最终操作函数:light_fresh->sampleLight_updateColor->hwLight_colorUpdate_HSV2RGB->pwmSetDuty(PWM_R_CHANNEL, gammaCorrectR * PWM_FULL_DUTYCYCLE);


zcl_onOffAttr_t *pOnoff = zcl_onoffAttrGet();
pOnoff->onOff = 0;
light_blink_start(5, 200, 200); //配网成功,闪烁5次,并关灯

## 色温
//Color temperature = 1,000,000 / ColorTemperature in the range 1 to 65279
#define COLOR_TEMPERATURE_PHYSICAL_MIN	147//6800K
#define COLOR_TEMPERATURE_PHYSICAL_MAX	312//3200K


控制色温:5000K(1000000/5000=200)
zcl color-control movetocolortemp 200 0x002
send 0xA89C 1 1

0x0300 ,0x0007	-> g_zcl_colorCtrlAttrs.colorTemperatureMireds
0x0300 ,0x400B  ->g_zcl_colorCtrlAttrs.colorTempPhysicalMinMireds
zcl global read 0x0300 0x0007 //读取当前色温
zcl global read 0x0300 0x400B //读取最小物理色温
zcl global write 0x0300 0x400B 0x21 {96 00} //设置最小物理色温0x0096(150)

## 色温转PWM代码
```c
色温转PWM
void temperatureToCW(u16 temperatureMireds, u8 level, u8 *C, u8 *W)
{
	zcl_lightColorCtrlAttr_t *pColor = zcl_colorAttrGet();
	*W = (u8)(((temperatureMireds - pColor->colorTempPhysicalMinMireds) * level) / (pColor->colorTempPhysicalMaxMireds - pColor->colorTempPhysicalMinMireds));
	*C = level - (*W);
}
void hwLight_colorUpdate_colorTemperature(u16 colorTemperatureMireds, u8 level)
{
	u8 C = 0;
	u8 W = 0;

	level = (level < 0x10) ? 0x10 : level;

	temperatureToCW(colorTemperatureMireds, level, &C, &W);

	u16 gammaCorrectC = ((u16)C * C) / ZCL_LEVEL_ATTR_MAX_LEVEL;
	u16 gammaCorrectW = ((u16)W * W) / ZCL_LEVEL_ATTR_MAX_LEVEL;

    pwmSetDuty(COOL_LIGHT_PWM_CHANNEL, gammaCorrectC * PWM_FULL_DUTYCYCLE);
	pwmSetDuty(WARM_LIGHT_PWM_CHANNEL, gammaCorrectW * PWM_FULL_DUTYCYCLE);

}
``` 

## 协调器开发(ZC)
  1. 配置HCI_UART管脚(sampleGw\board_8258_dongle.h)
    #define UART_TX_PIN         	UART_TX_PB1
	#define UART_RX_PIN         	UART_RX_PB0
  2. app配置(apps\sampleGw\app_cfg.h)
    #define	UART_PRINTF_MODE				1
    #define	ZBHCI_UART						1
    #define BOARD    BOARD_8258_EVK_V1P2//BOARD_8258_DONGLE
  3. 编译下载,并打开控制程序(zigbee gateway User Interface)
    1. 115200/n/8/1
    2. bdb -> startNetwork
    3. mgmt -> Permit Join:0xFFFF 0x60 
  4. 有设备入网后,按SW2 模拟发送开关指令(sampleGw\app_ui.c)
    s32 brc_toggleCb(void *arg)
  5. 网络参数配置
    APS_BINDING_TABLE_NUM           绑定表个数
    APS_GROUP_TABLE_NUM             分组表个数
    TL_ZB_NWK_ADDR_MAP_NUM          地址映射表个数
    TL_ZB_NEIGHBOR_TABLE_NUM        邻居表个数
    ROUTING_TABLE_NUM               路由表个数
    NWK_ROUTE_RECORD_TABLE_NUM      路由记录表个数
    NWK_BRC_TRANSTBL_NUM            广播表个数
    NWK_ENDDEV_TIMEOUT_DEFAULT      终端设备保活超时时间
    ZB_MAC_RX_ON_WHEN_IDLE          终端设备空闲时RF接收机的状态
    ZB_NWK_LINK_STATUS_PEROID_DEFAULT   link status命令的周期时间   

## 调试记录

//多于2个ep,需要修改ZCL_CLUSTER_NUM_MAX的数量

nwkSeqNum大概15秒更新一次
nwkSeqNum:F8,macSeqNum:54,frameRetryNum:3,panId:9EB0,nodeId:91FE,txTotal:0,txFail:0 
 046a94:  00 00 00 00 a9 69 11 88 64 38 c1 a4 73 91 4d 96 
 046aa4:  2b 38 c1 a4 54 00 8a c3 b0 9e fe 91 0f 00 f4 01 
 046ab4:  00 2b 00 00 00 22 96 35 e6 32 75 06 38 c1 a4 ff 
 046ac4:  ff ff 00 0f 00 00 00 00 00 0f d0 *54 03 20 04 00 
 046ad4:  00 00 00 05 08 00 1a 01 00 00 00 00 40 bc 03 00 
 046ae4:  0f 03 19 00 08 00 78 03 88 13 00 00 35 e6 32 75 
 046af4:  06 38 c1 a4 b0 9e fe 91 00 00 00 00 00 00 00 00 
 046b04:  *f8 02 8e 02 00 02 00 00 0f 82 00 08 73 91 4d 96 
 046b14:  2b 38 c1 a4 01 00 00 00 00 00 00 00 00 00 00 00 
 046b24:  00 88 10 02 00 70 ef 05 ff ff 02 00 00 00 00 00 
 046b34:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
 046b44:  00 00 00 00 08 00 01 00 01 03 03 03 00 0f 00 00 

zcl_seqNum
 uartRxFrame 串口接收buf
rf_rxBuf  接收的buf指针

0xA4C1382B964D9173 最少出现二次控不到,串口接收不到

## 禁止ZC变通道等能数,获取nwk key
https://developers.telink-semi.cn/topic/1335
可以配置以下宏定义
#define BDBC_TL_SECONDARY_CHANNEL_SET
#define BDBC_TL_PRIMARY_CHANNEL_SET
#define DEFAULT_PANID

读取该变量SS_IB().nwkSecurMaterialSet[1].key获取nwk key

OTA可以使用BDT或者Tools目录下的ZGC_NEW工具将OTA文件写入到协调器的flash，之后使用OTA 页面的notify告知待升级设备。


### 链接:
[高分辨率RGB LED 混色应用笔记]:http://www.microchip.com.cn/community/Uploads/Download/Library/00001562a_cn.pdf
[zigbee ZCL里的Hue和RGB值怎么转换呢]:https://www.sekorm.com/faq/53114.html
[RGB与HSV之间的转换公式及颜色表]:https://www.cnblogs.com/klchang/p/6784856.html


## IEEE地址存储
中控:IEEE Address response: (>)C3994D8158DCEA89
节点:node:[0x4D8158DCEA89C399]
flash: 076000:  89 ea dc 58 81 4d 99 c3 

IEEE Address response: (>)A4C138299952C053
node:[0xA4C138299952C053]
 076000:  52 99 29 38 c1 a4 53 c0

## E06
zcl color-control movetocolortemp 168 0x000
zcl color-control movetocolortemp 168 0x000
===cur:168,43008,step:0,+:0

zcl level-control o-mv-to-level 247 0
oriLevel:247,colortemp:168

zcl color-control movetocolortemp 25 0x000
===cur:167,42752,step:36352,+:0

zcl level-control o-mv-to-level 248 0
oriLevel:248,colortemp:167
==================发送以下序列即__tickCount = 1===================
zcl color-control movetocolortemp 0 0x000
zcl color-control movetocolortemp 0 0x000
zcl level-control o-mv-to-level 247 0


=======================================
#define MSG(format,...) \
    do{\
        char __buf[200];\
        mysprintf(__buf,format,##__VA_ARGS__);\
		drv_uart_tx_start((u8 *)__buf,strlen((const char *)__buf));	\
    }while(0)

MSG("%s\n",__FUNCTION__);

static void sampleLight_moveToLevelProcess(u8 cmdId, moveToLvl_t *cmd)
{
	zcl_levelAttr_t *pLevel = zcl_levelAttrGet();
    cmd->transitionTime = cmd->transitionTime * 10;


场景:sampleLight_moveToColorTemperatureProcess

开关
light_applyUpdate
light_fresh
  sampleLight_updateColor
    hwLight_colorUpdate_colortemperature