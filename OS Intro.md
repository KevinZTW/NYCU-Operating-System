# OS Introduction

### What's Operation System?

- A program act as an `intermediary` between a user of a computer and the computer hardware
- kernel + [[coreutil]] + binutil + ...... device drivers 是 kernel 的一部分
- 用戶的角度 : 方便大家好用
- 系統的角度 : memory cpu 等資源的調度、分配

### General Purpose (Goal) Operation System

Driver 之上有 OS, OS 是在 implement 一堆 API 來給大家使用 (systme library / system call) But when would we called the OS APi? when we compile => link the file, the system library would also be linked, then we would get the executable binary
![[Pasted image 20210412223757.png]]

### Four Components of a Computer System

![[Pasted image 20210915112254.png]]

1. User - people, other computers
2. Application -
3. OS - control and coordinates the use of hardware/resources
4. Hardware - CPU, memory

### Define OS

- Resource allocator : manages and allocates resources to insure efficiency and fairness (兩者難同時達成)
  - 時間相關的 : cpu 時間, I/O bandwidth
  - 空間 : 記憶體、磁碟
- Control Program (阻止不檔使用、一個程式竄改另一個程式的檔案 pointer 亂指)
- Kernel : 作業系統別名, 核心的意思 the one programm running at all times

### Goals of an Operation System

- Convenience 讓電腦系統方便使用
- Efficiency 有效率地利用硬體 (同時要兼顧公平性 fairness 及可預測性 predictability)

Two goals are sometimes contradictory

### Cont'd

- No Black and white definition
  - 廠商出貨時搭載的軟體就是作業系統? 那 browser?
- The one program running at all times on the computer is the kernel

  - Everything else is either a systme program or an application program

- UNIX OS 其實是一份標準只要符合這個 spec 就可以說自己是 UNIX
  - POSIX (IEEE 1003.1-2017)
  - 必須要有 A kernel 在 user program 之下、永遠存在的程式碼
  - 必須要有 A collection of systme programs ~= [[coreutil]] + binutil

## Pillar 1 : Process Management

- Process is a program in execution, program is passive entity, process is an active entity
- Process need resources like memory...
- OS provide synchronization mechanism (像是信號機、mutex) 以及 communication 像是 pipe 給 process 使用

## Pillar 2 : Memory Management

- Data in memory before and afrer processing (int arr[ ] 可讀可寫)
- Instruction in memory in order to execute (程式執行碼 mov cmp jmp....只可讀)
- paging...

## Pillar 3 : File System

## Computer - System Organization

- One or more CPUs device controllers connect through common bus porviding access to shared memory
- Goal : Concurrent execution of CPU and devices competing for memory cycles

### Computer - System Operations

![[Pasted image 20210609223025.png]]

- each device controller is in charge of a particular deivce type, each of them has a local buffer
- CPU moves data between memory to local buffer in device controllers

資料是要先寫到 Devices Controller 才會寫到 cpu

- Status reg :　告訴你現在現在再進行 io 還是在閒置
- Dara reg : 想像成另一個 buffer

### Buy/wait ouput

介紹最簡單的 io 控制方式, 簡單來說就是讓 cpu 去檢查 buffer 是不是滿了 滿了的話就等他, 沒有滿的話就搬資料過去

```C++

#define OUT_CHAR 0x1000 //device data register
#define OUT_STATUS 0x10001 // device status register

current_char....

```

## Interupt I/O

既然剛剛的方式很沒效率, 改成設計成隨時可以打段 cpu 來做另一件事情
cpu 做他的事情忙到一半, 突然被 interupt 來做另一件事, 做完再 resume 回去做它原來的事情

- hardware may trigger an interupt at any time by sending a `signal` to cpu
- software may tigger an interrupt by
  - error (division by zero / invalid memory access)
  - equest for an operating system service `system call`
- software interrupt also called trap
- interrupt architecture must save the address of the interrupted instruction
- incoming interrupts are disabled while another interrupt is being processed to prevent a lost interrupt (後面在傳來的 signal 會當作沒看見)
  ( 用來避免 一再一再的記住不同的狀態 來回復到最初打斷前的狀態)
  (e.g.動滑鼠沒反應可能是因為他卡在某一個 interrupt 裡面了)

#### hardware Interupt

hardware send out a siganl (which would be first receive by mother board) and called the OS

1. check interrupt vector (kind of an array) (which contains the address (function pointer) of all the service routines
2. perform the service routine for device i (which record on the interupt vecotr)

#### software interrupt

## Storage Device Hierarachy

video : 4-c

講述儲存設備的快慢, ram , sram, 快取, 快取下資料一致姓

## Hardware Protection

video : 4-d

### Dual - Mode Opertaion

- Provide hardware support to differentitate between at least two modes of operations
  用來去區分 OS 跟一班的執行程式避免天下大亂

1. User mode
2. Monitor mode(also called kernel mode or system mode)

- 要教 OS 幫你做任何事情一定要透過 `system called` 而這個 call
  一定要透過 `interrupt` 來送出

- 因此在硬體上就會設計成, 平常是 1 usermode, 而當他偵測到 system call 的時候就會 flip 成 1 kernel 因為她知道 interrupt 執行的就是 OS , 因此 cpu 透過這個 bit (0,1) 就知道現在是哪一個 mode

- 在 cpu 指令集會有 Privileged instructions
  - 他會檢查, 一定要是 monitor mode 才幫你執行, 發現不是的時候會丟一個 interrupt 出來交給 OS 處理

## I/O Protection

- All I/O instructions are privileged instructions
  -     因為他們全都是共用的, 螢幕滑鼠網路卡.....如果大家都可以寫入的話會天下大亂
  - 全部透過 OS 才能對他們操作
- 駭客還是會透過其他方式, e.g. memory content 去修改掉記憶體中 interrupt voctor 裡面的值

### hardware address proctection

每一個程式都會有 base register 記憶體分配開始的位置 跟 limit register 結束的位置, 每一個 cpu instruction 都會先去檢查是否在這個範圍裏面

### CPU Protection

為了防止 cpu 被某一個程式霸佔, cpu 有一個 timer 機制, 時間到的時候就會 interrupt 改成執行 OS, 由 OS 來決定是否還是剛剛那個程式繼續執行
