# OS IOs

| [[CMU 15-213  Lec16 System Level IO]]

- Operation System is called syschronously by user programs, 但是它同時也會被 triggered by asynchronous events i.e. hardware events from outside of the CPU

- IO device 跟 CPU 是平行在跑的，要顧慮好誰要等誰，才能最優化 CPU utilization 以及 I/O latency

### Typical PC Organication

- North Bridge (北橋 memory hub)
- South Bridge (IO Hub)

以下這張圖是以 x86 作為栗子
![[Pasted image 20210917153947.png]]

- 高速裝置 Memory Hub
  這些裝置傳輸量很大
- 其他的部分稱作 I/O Hub

  - IDE disk controller : e.g. SATA
  - expansion bus interface : e.g. USB

- 注意 IO 跟 cpu 是同時在進行的！

### I/O Hardware

- Common concepts
  - Host controller 驅動程式對他下命令
  - Bus or interface (Host 透過他對 Device controller 溝通)
  - Device controller
- I/O instructions contorl devices
  - Direct I/O
  - Memory -mapped IO

#### Device I/O Port Locations on PCs (partial)

![[Pasted image 20210917154507.png]]

- 注意這邊的是 Direct I/O ，會有一個 I/O address space
- 舉例 timer 非常重要 (timer interrupt / timer tick) 如果我想要設定他兩秒鐘後發出中斷，那端程式可能會像是 (這段是給我一個感覺底層是怎麼運行的)

```
		mov al, 0x36 // 存到暫存器 al
		out 0x43, al //

		mov ax, 11931
		out al, ah
		out 0x40, al

		//Channel 0 0x40 System Timer
		//Channel 1 0x41 DRAM refresh (obsolete)
```

這些事情都是在作業系統 kernel mode 底下才可以進行，我平常開發應用是無法執行的
這些是 privilege instruction

### I/O Operation 兩種方法

- I/O devices and CPUs execute concurrently

  - Polling I/O : The CPU waits on an I/O
  - Interrupt I/O : The CPU will be notified

- Polling
  ![[Pasted image 20210917155437.png]] - Busy waiting 很忙碌，忙著在等所以 cpu 鎖死了，他一直問 data 好了嗎？read 好了嗎？

## Interrupt

中斷（英語：Interrupt）是指處理器接收到來自硬體或軟體的訊號，提示發生了某個事件，應該被注意，這種情況就稱為中斷

### Hardware Interrupts (IRQ interrupt request)

可參考 : [[Signal]] | [[ECF Exceptional Control Flow 異常控制流]]

- Hardware 透過送出信號來 interrupt
  OS 的概念要學的好，要把 interrupt 搞懂一點 OS 是圍繞著 interrupt 在設計的

- ![[Pasted image 20210917160532.png]]
  pic 是中斷控制器 (本身也是一個硬體，前面有看到 interrupt controller)

1. CPU 發出命令給 I/O device
2. I/O 完成 送了一個信號給 PIC (因為可能同時有一籮筐 IO 裝置送信號，透過他可以做一個優先排程)
3. PIC 送給 CPU 信號 可以看上圖 memory 的部分，上方是低位址，在低位址 的部分有一個 IVT 中斷向量表，收到的信號會帶著一個數字 123

先把目前的 register, stack pointer 暫存起來

4. CPU 做的 CPU 的==硬體== 會去 IVT interrupt vector 查表 123 拿到記憶體位址並跳到該處，也就是對應的 ISR interrupt service routine (應用中斷程式) 的位址最後會中斷返回到 user program 中

- 特別之處是我沒辦法預測什麼時候會產生，硬體高興傳給我就傳給我，隨時都可以進來

### Software-generated interrupt : Trap

- devide by zero, acces violation, system call
- segment fault, core dump

會根據我所產生的原因 `CPU` 會根據產生編號去製作編號
跟剛剛的差別是信號不是外面傳進來的，我執行程式的時候碰到非法位址 CPU 會

### software interrupt vs hardware

from geeksforgeeks

SR.NO. Hardware Interrupt Software Interrupt
1 Hardware interrupt is an interrupt generated from an external device or hardware. Software interrupt is the interrupt that is generated by any internal system of the computer.
2 It do not increment the program counter. It increment the program counter.
3 Hardware interrupt can be invoked with some external device such as request to start an I/O or occurrence of a hardware failure. Software interrupt can be invoked with the help of INT instruction.
4 It has lowest priority than software interrupts It has highest priority among all interrupts.
5 Hardware interrupt is triggered by external hardware and is considered one of the ways to communicate with the outside peripherals, hardware. Software interrupt is triggered by software and considered one of the ways to communicate with kernel or to trigger system calls, especially during error or exception handling.
6 It is an asynchronous event. It is synchronous event.
7 Hardware interrupts can be classified into two types they are: 1. Maskable Interrupt. 2. Non Maskable Interrupt. Software interrupts can be classified into two types they are: 1. Normal Interrupts. 2. Exception
8 Keystroke depressions and mouse movements are examples of hardware interrupt. All system calls are examples of software interrupts

### 自己補充 signal

訊號/信號是 Unix、類 Unix 以及其他 POSIX 相容的作業系統中行程間通訊的一種有限制的方式。它是一種異步的通知機制，用來提醒行程一個事件已經發生。當一個訊號傳送給一個行程，作業系統中斷了行程正常的控制流程，此時，任何非原子操作都將被中斷。[^1]

[^1]: https://petertc.medium.com/session-process-group-and-signal-in-linux-7fbe85c0b0c5

### Direct Memory Access

![[Pasted image 20210922103336.png]]

IO 的最後一步資料如果很大，DMA 就是一個特殊的硬體，負責把 IO 資料一大塊一大塊搬到主記憶體裡面，每次搬完就產生中斷

- 減少中斷頻率
- 讓 CPU 不用花時間班資料
- 可以一次處理好幾個 IO 裝置

### Synchronous I/O v.s. Asynchronous I/O

軟體層面的一個概念，跟前面的中斷 / pulling 不必然是有關的

- 下面哪些是 sync / async
  - fsync() sync, 搭配 fwrite 使用的 syscall ，把記憶體裡面殘存的資料在那邊等直到強制全部寫出去到 IO 裝置裡面，當這個 function return 的時候資料都確實寫入了
  - fread() sync, 從 IO 裝置讀資料進來，資料沒有讀進來程式應該是會需要阻塞的，blocking IO
  - fwrite() async, 為了增加效能，不會 blocking，作業系統自己會找時間寫出去
  - aio_read() async, read 一般來說都是 sync 但是這個比較特別不會卡，用在 prefetch，預先把之後會用到的檔案從 IO 裝置讀進來到磁碟 cache 真的要 read 的時候有讀到就很好沒有就算了

![[Pasted image 20210922103739.png]]

## Multiprogramming

- Multiprogramming 的概念就是，記憶體裡面同時載入了好幾隻程式
- single process can't let cpu and IO busy all the time
  \-------------------
  \| OS | job 1| job 2 |...
  \-------------------

### Timesharing

- a logical extension in which CPU switches process so frequently that user can ineract with each job while it is running, creating interactice computing
- 我可能會一次開好幾個程式，同時希望他們都可以有一點進度
  - switching frequency usually >= 100 Hz 一秒鐘換 100 次
  - Each user has at least one program executing in memory (ch3 process)
  - If several jobs ready to run at the same time => CPU scheduling (ch5 )

### Terminology

- Mutliprogramming 同時載入多個城市來執行
- Multitasking 沒有特別講怎麼切換, 也許有 io 才換、有人 sleep 才換....
- timesharing : 定時的換 multitaksing + periodic switching among process