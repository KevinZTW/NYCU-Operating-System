# OS Process

可參考 : [[Berkely CS162  Processes Fork I O Files]]

- Process 是一個跑起來正在執行的程式 Program，Program 是靜態的程式碼

  - 基本上 chrome 開一個新的 tab 就會多一個 process 我們稱之為 instance

- job / task / process 指的是同一件事情，有時候會混著使用!

### stack/heap/data

可參考 : [[CMU 15-213  Lec 5 Machine Level Prog amming]]

![[Pasted image 20210929105459.png]]
回顧一下 可以看到有 `data` 放的是變數 `text` 是 \_ start 以下的程式碼

![[Pasted image 20210929105617.png]]
注意記憶體位址是 0 到 max (因為是虛擬的記憶體空間)

- Process includes
  - text
  - stack
  - data
  - heap

```c
int i;
int foo(int x)
{
	int *y;
	i = 0;
	y = (int *)malloc(100);
}
```

- 以上代碼若編譯成二進位的話，各類會被放在哪裡 ? 編譯完變成 mov, add.... 會在 text

- global 變數 i 在 data

- 老師說 c 語言 local variable 都是放在 stack 上面
- stack 上放 activation record 每次呼叫一次就會跑出來一個 在這裡的情況包含 x, \*y, return
- malloc 是動態配置的記憶體，放在 heap 裡面

### Process State 狀態

![[Pasted image 20210929112752.png]]

- new :
- running : 單 CPU 下一次只會有一個 process running
- ready : process 準備好了，等著 CPU 來執行它
- waiting : 沒辦法繼續執行，正在等待 (sync IO e.g read() 因為 async IO 不用等) 既然沒辦法執行你必須把 cpu 交出來
- terminated :

- 比較需要注意， interrupt 打進來的話會從 running 跑回去 ready ，這邊通常是 timer 發出的 參考: [[ECF Exceptional Control Flow 異常控制流]]
- waiting 的 process 等到事件發生時是回去 ready 排隊

蠻有趣的，老師說 software-generated interrupt = trap 不一定會讓 process 從 running 轉換到 ready 實際上還是要看他做了什麼事情，如果是呼叫一個 system call 進到 kernel mode 去做一些事情這個過程我都還是 running 的，所以實際上還是要看 trap 之後做了什麼事情

老師的題目 : hardware interurpt can trigger

- running -> ready O timer 發出 interrupt
- running ->waiting
- waiting -> ready O 本來在等 IO, IO 做好了 (IRQ) 發出 interrupt 中斷請求（interrupt request，IRQ

- starting sync IO : run -> waiting
- process resume :
- process suspend

### Process Control Block (PCB)

![[Pasted image 20210929114914.png]]

- 這個 PCB 是放在主記憶體裡面
- process `state`
- saved CPU registers value

### Context Switch

- context switch 為了效率通常用組合語言寫
- hardware 要花時間 software 塞滿的 pipeline, 建好的 cpu cache 都要丟掉

老師以一個簡單的 OS 舉例，說 linux 流程也差不多
![[Pasted image 20210929115928.png]]
SS:SP 是 stack pointer
CS IP 是 program counter
失去 cpu 時 :

- 首先 把 cpu 暫存器的內容存到 stack 上
- stack pointer 放到 PCB 裡面
  得到 cpu 時
- 從 PCB / TCB 中把 stack pointer 載回來
- 把剛剛暫存的寄存器內容都做恢復

[[OS context switch]]

### Schedulers

- 呼應前面說，這個就是一個 policy 選哪個都可以並不會影響`正確性` 只會影響 `效能`
- 做的事情其實就是從 ready queue 裡面釣出一個來執行，

#### short-term scheduler (=CPU schedulers)

- invoked very frequenlty (millisceconds) => (must be fast)
- 更換的時間包含 context switch + 做決定的時間，分秒必爭

#### Long-term scheduler is invoked 非常不頻繁

我有一大堆的工作，他挑幾個載入到記憶體裡面去，剩下的在..... 硬碟? - 他控制了 degree of multiprogramming 同一時間載入了多少 process 到 memory 裡面 (我 memory 裡面足夠放十個 process 就放十個來執行) - 需要適當的混合 IO bound process 以及 cpu bound process - IO bound 循環應該是 ready -> run -> wait -> ready.... - CPU bound 是他 run run run 太久被踢出去(time slice 到了) ready 再 run run run 壓縮編碼打 game

為什麼我對 long term scheduler 這麼陌生? 因為 time sharing machine (像是我們一般人用的電腦)是沒有的，這種情境下 degree of multiproamming 是取決於硬體限制 (ram 多大..)

#### Medium Term Scheduling

- 想要協助讓我們可以一次跑更多的 process, increase the degree of multiprogramming
- 他會去看哪些 process 是 inactive 的把他 swap out 存放到外部的磁碟

![[Pasted image 20211001154809.png]]

## Operation of process

### 物理記憶體 layout

![[Pasted image 20211001155307.png]]

### 建出 process fork ()

- unix 中可以用 `fork()` 生出一個 child, `exact copy of parent` 兩者`獨立`、內容一模一樣
- 所以如果原本有一個變數 x 在 child 中 ++ 並不會影響 parent 的數值

vfork 和 fork 的最大區别就是共享 mm 的資源，隻要其中一方修改 mm 的資源，另外一方就會看到

### Tree of processes in Linux

![[Pasted image 20211001160545.png]]

- init process 會幫我們生出其他這些 process

### Terminate process

- 自己呼叫 exit 這個 system call 這個 process 就被結束掉了，
- orphan process : is a computer process whose parent process has finished or terminated,
- zombie process 已經結束 `memory` `pcb`已經釋放掉了但是在那邊等 parent 接收 process table 中 他的 return value ，如果 parent 一直沒有執行 `wait` 就會一直在那邊，很多 zombie 的時候會把 process table 佔滿最後導致沒辦法產生新的 process

- 如果 parent 結束我卻還沒結束我就會變成 orphan 被 `init` 收養，因此可以透過這種手法把 wait 交由 init 處理

作業手法 :

1. 註冊 sigchild
2. detach (double fork)
   ....

### fork() then exec()

- fork 再 exec 有點弔詭? 好不容易 copy 出來 (parent 用了 400mb 空間就要整個複製給小孩?)，結果 exec 整個又把全部記憶體空間清掉 (data, text....)
- 其實父親跟小孩是指向同一塊實體記憶體 (透過 memory mapping 這個要 CPU 中的 MMU 來幫忙)，誒這樣跟剛剛說的又不同啊不是說好兩者獨立修改不互相影響？=> 這邊用了一個技術 copy on write 有修改的話我再幫你生一份新的改動資料

- 如果 CPU 沒有 MMU 就要改用 `vfork()`嵌入
  - parent 會先停下來不執行把 stack data text 借給 child 來用
  - 接下來 child 可以執行 `exec`
  - 這個時候 child 去更動全域變數 x++ 因此 parent 回來之後那個數值就被改動了

## 問題

program counter 是 作業系統實作的嗎?
目前想像應該是 ISA instruction set architecture 所識做的，計算機結構之後可以問

### Communicatino Models

![[Pasted image 20211006103236.png]]

#### Message passing

以前說到的 micro kernel 就是屬於 message passing

- 方便 自帶同步機制
- Message 效能比較不好，因為要 copy overhead, user-kernel switch overhead
- pipe /socket 就是一個栗子

普通的 pipe / name pipe
普通的 pipe
同一個 process 對一個 pipe 只能是單向的操作，那作業系統怎麼知道你這端視要讀還是要寫呢？ => close 你不要的那端
要有一個觀念， pipe 是在 kernel 裡面的！讀的人是去那邊看看有沒有資料在 buffer

```c
int fd[2], nbytes;

```

#### Shared Memory

有觀念要建立是記憶體映射的事情，這邊
attach - 把建立的共用記憶體映射到我自己的記憶體空間，之前 fork child 也是用了一樣的方式

shared memory 好處是不用 copy 資料
壞處是不具備同步的能力，放的時候空間滿了 kernel 不會幫你解決

### Producer-Consumer Problem

- paradigm for cooperating process, producer process produces info that is consume by a consumer ( run concurrently)
- objective : to sunchronize a producer and a consumer via shared memory
- issues :
  - the buffer size is limited
  - overwriting and null reading are not allowed

### Bounded Buffer - shared memory solution

一個環狀的 buffer , producer

- shared data

consuemr 怎麼判斷拿到空的東西 ?

- 兩者在同一個地方? 但是這樣如果一路放過去還是會重疊
- 改成 p 只會放到 c 的前一位

```c
//一直繞 有底類似 pulling
while(true){
	// 會先檢查 buffer 是不是滿的
	while ((in+1) % BUFFER SIZE count == out)
		;//do nothing
	buffer[in] = item;
	in = (in + 1) % BUFFER SIZE;
}
```

```c
while (in == out)
	; //do nothing
	//remove an item from the buffer
	item = buffer[out];

```

### Bounded Buffer Problem

- data corruption?
  為什麼不會出問題? 因為兩者操控兩個不同的指標，而且都是拿完資料放完資料後才去改數值

- performance issue?
  - yes 當 consumer 發現裡面是空的時候他就會一直繞，他會一直霸佔 cpu 不讓給 producer 嚴重 performance 問題！
- why not to use a free slot counter?
  - 如果我改成用一個 free 如果 free = 0 表示 full free = N 是 empty
    => race condition

如果去採非法記憶體，會被 CPU 的 MMU 抓到，產生一個 trap 到作業系統並且送出一個 signal SIGSEGV
