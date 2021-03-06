/*
  《STM32顶层逻辑控制模块》
  PART1 ：串口UART
  在main.h文件中配置与DSP通信的串口为UART4，对应的单片机引脚PA0:Tx，PA1:Rx
  在main.c文件main函数中对串口进行实例化配置，需注意的地方是配置的波特率是115200，因为DSP端配置的是115200，需与之一致
  最后初始化串口，即开启串口，这里开启串口会导致耗电，所以在后面的任务中可以在等待任务期间关闭串口（HAL_UART_DeInit），在开启任务时再打开串口（HAL_UART_Init）
  
  PART2：TIM时钟
  时钟用于产生计时，可以通过宏 SECONDS_PER_TIM 定义每个时钟的周期秒数，即每SECONDS_PER_TIM 秒产生一个时钟中断，执行中断服务函数HAL_TIM_PeriodElapsedCallback
  在中断服务中对当前时间等计时变量进行更新，并判断是否到达任务开启条件，即累积中断次数是否到达设定次数
  计时变量由结构体CurrentTime 的对象表示，包含day,hour,minute,second，每次中断则将时间更新累加SECONDS_PER_TIM 秒；当前时间变量的作用是用于告诉DSP当前时间
  为了防止CurrentTime的对象变量在复位后丢失变量值，需将该变量声明为非初始化内存区，定义在特定的内存地址中。
  
  PART3：看门狗WATCHDOG
  看门狗可以看做是STM32外部的设备，其作用是在程序运行期间进行倒计时，主程序可以随时对看门狗进行feed，若在设定的时间内看门狗没有得到feed则看门狗负责将STM程序重启
  若在设定时间内看门狗得到了feed，则倒计时重新开始计时。feed操作通过函数FeedWDG() 完成，看门狗的倒计时时长通过宏 WDGTIME 定义，单位是s
  
  PART4：主函数main
  main函数的最开始需要对程序运行状态进行判断，即此次进入main函数是初次进入还是复位后进入，若是首次进入则对几个
  定义在非初始化区域的变量进行初始化，否则进行变量值更新。
  主函数中首先进行串口，时钟和看门狗的配置和初始化工作,然后进入while循环，执行业务逻辑，此业务逻辑即单片机在长期任务中所处的状态，主要过程如下：
  
  1. 在while循环中等待waiting_status状态量变为false，否则一直在循环中feed看门狗（因为不feed的话20s后看门狗就超时了，会复位重启程序，在程序运行的过程中需要时长记得去feed看门狗）
  waiting_status状态量在TIM时钟的中断服务函数HAL_TIM_PeriodElapsedCallback中被改变，当TIM中断次数到达指定的次数，则修改这个状态量为false，则在中断函数结束后单片机返回到主函数中时，
  while循环结束，开启一次新的探测任务
  2. 探测任务是通过STM32单片机与DSP通过串口通信的方式进行，单片机通过串口发送ATCMD给DSP，DSP完成指令后返回完成状态，单片机成功接收到这个返回状态后则继续下一步指令，若
  返回的状态不符合预期则任务当前指令出错，需要进行相应的处理，这个错误处理是尚待开发的
  首先DSP通过控制GPIO引脚PE0,PE1,PE2,PE3的电平将继电器的常开接口闭合，从而使得DSP与发射板上电，单片机等待几秒钟（这个时间需要测试，因为DSP上电后初始化需要一点时间）
  然后通过ATT()函数对DSP进行通信测试，若测试成功则可以进行后面的指令任务发送，否则进入错误处理（可以选择等待几秒再尝试ATT()或其他操作）。
  后续的指令操作可能包括：监测电池电压，根据电池电压的剩余量控制今后的任务周期；发射探测声波；存储回波数据到SD卡；等等操作
  最后结束了一次探测任务，通过GPIO引脚控制继电器关闭DSP和发射板电源，然后进入while()等待下一次任务。
 */
