# OS 架構

# System Structures

- System Calls
- Operating System Structurede

### 作業系統提供的服務

![[Pasted image 20210922110551.png]]
POISX spec : UNIX = kernel + sys programming (binary utility / core utility)

- system program : ls, mv....
- 使用者介面
  - CLI command line interface :
    - shell refers to interface program between users
  - GUI : many systems now include both CLI and GUI interfaces

### GUI

PARC 研究機構發明了 GUI 除此之外還有 mouse, laser printers, ethernet, small talk
GUI 也是一種 shell

### System Programs

還包括在 compiler e.g. gcc

## System call

- 為了移植性好、開發簡單，還是用 c library / api 的多

| Application  |            |
| ------------ | ---------- |
| Library      | c library  |
| API          | 更底層一點 |
| System calls |            |
| Kernel       |            |

寫程式可以選擇 c library, API, Sys call 但是一般人主要都是呼叫 c library

System call is programming interface to the services provided by the OS kerenel 給我寫程式的時候呼叫用的，但是一班不這麼做

Mostly accessed by programs via a high level Application Program interface (API) rather than direct system call use，typically written in a high level language (C or C++) Portabiliy and simplicity 移植性跟簡易性，因為 kernel win10, win11 kernel 可能不一樣

| C : fopen("w+"....)     | C language |
| ----------------------- | ---------- |
| WIN32 API: CreateFile() | c library  |
|                         |            |
|                         |            |

#### Example of Standard API

- WIN 32 ReadFile()...跟鬼一樣

### System Call implementation

要呼叫作業系統需要一個 Trap

範例呼叫 sys call

```assem
section .data
msg db "hello world!", 0xa	//我們的 string
len egu $ - msg 			//定義時期的假變數，我們 string 的長度

section .text
	gloabl _start //導流，告訴程式這裡是進入點

start :
//write Hello World string
	mov edx, len
	mov ecx, msg
	mov ebx, 1		//first arg : file handle (stdout)
	move eax 4		//system call no.4 (sys_write)
	int 0x80		// call kernel (trigger a trap)
// and exit
	mov ebx, 0		//first syscall args:exit code(程式的回傳值，之前有提到為了等待子程式的回傳值會變成 zombie)
	mov eax, 1		//system call number
	int 0x80
```

- 呼叫作業系統是用中斷的方式
- 一但呼叫了 kernel， CPU 會去查剛剛的中斷向量表 (用剛剛的 0x80)
  ![[Pasted image 20211024212722.png]]

### syscall 簡介

![[Pasted image 20211028145426.png]]

用戶 prompt user string, save the string to disk file and terminate itself - prompt user string : since in linux stdout, stdint implemented as file,we need to use syscall `write` to write the string to `stdout` - save the string to disk :as above, use syscall `sys_read` to read input from `stdin`, and then since we need to save it on the disk so `sys_open` some file on disk and `sys_write` to it, and `sys_close` the file which would flush the stdout - terminate itself : use the `exit` syscall to terminate the process

     - `sys_gettimeofday` 得到時間, `sys_fdatasync` 跟 `fsync 很像`, `sys_waitpid`, `sys_fork` (這邊要複習作業一)

### Dual Mode Operations

![[Pasted image 20210922115045.png]]

為什麼 Linux Windows 會有 Dual Mode ? 因為我們不信任上面的軟體會有 untrusted program，因此一些比較敏感的操作保留給 kernel ，而 embeded system 為了效能、記憶體考量、CPU 支援能力、而且比較沒有 untrusted program 的問題

- 中斷有兩種，硬體的 IRQ 或是軟體的 trap ，觸發之後控制權移交給 kernel 進到 kernel mode
- user mode => kernel mode
- 進到 kernel mode 所有的記憶體，IO 裝置都可以使用可以操作

### More on systme calls

- for modern intel /AMD CPUs , SYSENTER/SYSCALL 建議使用
  本來是 INT 去查 IVT 中斷向量表，再去到 system call 的位址，現在直接把這個位址存在暫存器，這個 function 會直接過去查省略了 IVT 查表的部分

## Operation - System Structures

- 作業系統目的及 spec 影響他的 design

  - spec : server 不用考量電量，但是手持裝置就需要考量用電量、ram 大小
  - 目的 : server 需要考量運算速度，手持裝置考量圖形互動、續航力

- Design issues for different types of systems
  - Real-time OS : time predictability, low latency, romability 不能因為消能犧牲可預測性
  - Mainframe OS : thorughput, scalability
  - Desktop OS :
- Important principle to sepearte

  - Mechanism : how to do it, 怎麼樣去從磁碟上讀寫，基本上是不變的
  - Policy : What will be done 需要一些智慧，影響效能不影響正確性，哪一個 disk I/O operation 要先做?

  - which one of the following are policies, which are mechanisms?
    - process suspend : 沒有任何智慧，照做，mechanism CPU -> I/O
    - allocting the smallest among the memory blocks which are larger than the requested size : policies 我有三塊 100 kb, 50 kb, 1 MB 我要切哪一塊給 90 kb 的 request? (best fit)
    - marking a disk block as allocated : mechanism
    - servicing the disk I/O request which is closet to hte disk head : policies

## MS-DOS

![[Pasted image 20211001101108.png]]

- 可以說是沒有架構，沒有區分 user mode kernel mode 為什麼呢? 因為那個時候使用的 8086 CPU 還沒有辦法支援！ (後來是因為 cpu 有一個...privilege 的...腳位? 才有辦法做出區別)
- 下一層裡面還有一些特殊的程式 resident system program 像是計算機...
- 一次只能執行一個程式

## UNIX - monolithic

![[Pasted image 20211001101816.png]]

- 受限於硬體的功能，僅有很有限的架構 (user + kernel mode)
- The Unix OS 由兩部分構成 system program (binutils + coreutils : ls mb cp) + kernel
  - kernel 提供記憶體排程, 檔案管理.....
- 好處 : 提供基本的保護機制，把 user mode kernel mode 分開，且因為全部東西都包在一起執行效率高
- 壞處 : 裡面的架構糾纏不清，要學 kernel programming 極難入手牽一髮動全身，而且比較容易受攻擊 kernel 裡面有個洞就可以鑽進去竄

## Modules

- 通常用在一些 driver ，你用了一個新出的老鼠，作業系統就要去找相對應的 kernel modules 載入到 kernel 裡面
- 可以自己寫一段 module 一個 hello world 也可以，有 root 權限就可以把編好的 kernel modules 塞到 kernel 裡面

```
insmod ./hello.ko
rmmod ./hello.ko
```

## Micro-kernel

![[Pasted image 20211001102800.png]]

- 沒有必要的話就把 kernel 裡面的東西移出去到 user mode e.g. file system
- 只留下了 CPU 排程 , 記憶體管理....等很少的東西
- 把那些移出去的 file system device driver ....各自跑成一個 process
- 要讀檔案變成 application program 要先到 kernel 再到 file system 再到 kernel 再到 device driver 再到 kernel 再到硬體....再回去
- 優點 :

  - 比較安全 kernel 裡面的東西少少的 鑽進去也不太能幹嘛
  - 比較可靠一個 file system 掛掉重新開啟就好不會像 windows 變成藍色螢幕
  - 比較好擴展，每個 module 的 interface 都有定義好
  - 好 port 到各個硬體 : 每個 iot device 有的 hardware 不一樣，我只要調整那些 service process 就好

- 缺點 performance 很差 :
  - 每次訊息溝通都要在不同的 process 之間做複製
  - 還要在 user/kernel mode 之間切來切去
  - context switch 也會因此變得很頻繁 (我的應用切換到 file system process 切換到 device driver process.....)
- Windows NT 3.5 就有嘗試把 GUI 放到 user space 結果效能很慘烈

- linux 是單體式 mono...的
- andrew tanenbaum 也有設計一款 micro-kernel 的 minix (unix)
- intel ME 偷偷藏了一個作業系統在 cpu???
- Mac OS 是一種變形的 micro kernel - 裡面有一個 micro kernel `Mach`
  ![[Pasted image 20211001105222.png]]

## Virtual Machine

### native execution

![[Pasted image 20210929101701.png]]

- vmware 這種執行方式稱作 native execution，因為 host os 跟 guest os 的 ISA 都是一樣的 maybe x86
  - 虛擬機發出的非特權指令會直接交由 cpu 上執行 ( mov, cmp, ....)，但這樣到底哪裡有虛擬化?
  - 關鍵在於從 virtualization layer 往上 ，guest os 整個是以 `user mode application` 執行，當執行到特權指令 (需要 kernel mode 才能執行) 會被 trap 到 virtualization layer 這邊會做各種欺騙的動作讓 guest os 以為他摸到硬體了

### 好處 / 壞處

- protection ，memory space, CPU space, storage space 彼此都是獨立 isolation 的，看不到/不會影響其他機器的內容，但是也是麻煩的地方就是資源沒辦法互通

- 作業系統研究很常使用，開機很快，研究所操作的不會影響到正常日常使用
- 虛擬機的技術 197x 就有了，近年 cloud computing 加上老舊 server 不好管理而興盛 (因爲跑在雲端的是虛擬機，給他的資源可以伸縮，也可以容易的搬遷他)

- 虛擬層非常難實作，硬碟滑鼠...硬體各種操作 timing 都要忠實呈現模擬一套假的
- 效能也是個問題

### Java Virtual Machine

![[Pasted image 20210929103502.png]]
`Non-Native execution`

- Java 執行的指令有點像跑在一個假的 cpu
- Java 寫完會 compile 成 Java byte code，這個 byte code 會用軟體的方式間接做像是下面的執行
  ![[Pasted image 20210929103810.png]]
- Java 有出 just in time compiler 把他的程式碼編譯成 x86 的機器碼
