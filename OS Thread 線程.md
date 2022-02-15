# Thread 線程

[[OS Process & Thread (user kernel) 筆記]]

![[Pasted image 20211006113742.png]]
以前傳統的 process 只有一個 thread

A thread is a basic unit of CPU utilization; it comprises a thread ID, a program counter, a register set, and a stack 並與其他同一 process 的線程共享 data section (全域變數)、text (code section 程式碼)、stack、heap and other operating-system resource e.g. `signal` and `open files`

thread 比較奇妙的是 code, data, files 是共享的，修改彼此都看得到

- 很方便東西往全域變數放就好不用去搞 shared memory / message passing
- 每個 thread 都要有自己的 stack，CPU 暫存器也是互相不干擾的
- 老師說 local 變數是彼此看不到的 => 因為各自有獨立的 stack

可以想成進程是資源分配的基本單位，線程是系統調度的最新單位

當我們想要同時在同一個 application 處理許多事情 (browser 展示 UI 同時下載檔案，或是後端 server 服務很多個 client) 在以前 thread 還不風行的年代使用多個 `process` 是個常見方案，但是 fork 一個 process 耗費力氣成本，於是出現了 thread

課本提到 RPC server 通常是使用 multithread

### 好處

- Economy 效能省 : Thread 的 context switch 跟 creation 時間都比 process 快
- Responsiveness : 一個 thread 去處理 UI 另一個 thread 去在背後處理運算
- Resource Sharing : thread 會共享記憶體空間

![[Pasted image 20211006114530.png]]
注意到 memory 是同一份

## 歷史沿革

### Multithread programming 多核心編程

隨著硬體架構從單核心轉到多核心，Multithread programming 是一個可以更有效利用多核心以及優化併發的方式。

注意平行處理 (parallelism)是兩台咖啡機兩個隊列，兩者同時間在做運算處理，而 concurrency 併發則是大家各自都得到一點進度，也許是輪著跑像是一台咖啡機兩個隊列

再多核心上開發有以下難點 :

- Identifying task : 哪裡可以切成 concurrent 的任務？
- Balance : 切分後 equal value / equal work?
- Data splitting : 資料也要在不同的核心上處理？怎麼切分
- Data dependency : 一個任務資料相依到另一個 (e.g. merge sort)
- Testing / Debug

### Type of Paralleslism

- Data Parallelism : 都是矩陣乘法，只是每個人負責的資料不一樣，你 1~5 行...
- Task Parallelism : 你下載，他做 UI

## Multithreading Models

![[Pasted image 20211006115634.png]]

分為不需要 `kernel `支援的 `user thread` 以及由作業系統支援的 `kernel thread`，並非所有的作業系統、作業環境支援 multithreading!

- user threads are supported above the kernel
- kernel thread kernel / CPU 的 scheduler 是否看得到我有三個 thread ? 不一定 thread 是 user space 擴充的一種概念

### Many to One

- Many user-level threads map to single kernel thread

![[Pasted image 20211006115756.png]]

- kernel 並不了解你這個 Process 裡面還有其他的 thread
- user space 邏輯上你是有很多條 thread 但是 kernel 看不到不懂 user thread，他就覺得你是個 process，排程就根據 process 來排
- 因此在 user space 我要自己有一個排成器去分配資源給 thread (thread library 要自己處理)
- 為什麼要有這種彆腳的 model ?
  - 考慮到移植性 portability 因此先假設 kernel 看不懂 decouple with kernel
  - 但是這樣無法利用到 cpu 多核心 cpu1 running, cpu2 idle
- not mp-aware
- not trully concurrent

  - 如果一個 thread 去呼叫 wait 那大家都要等他，因為對 CPU 來說這整個就是一個 process
  - 在這種模式呼叫 read 就要很小心，有繞過去的辦法比方說改呼叫 ssize_t pth_read(int fd, \*buf, size_t nbytes) =>For GNU portable threads (M-1) only

- Examples :
  - Solaris Green Threads (for JDK options : green or native)
  - GNU portable threads ( An implementation of the Pthread specificaiton)
- 用 thread library 的時候要小心，自己去注意是多對一還是一對一還是....要如何判斷? 開很多個線程其中一個去呼叫 read 等阻塞的指令

### One to one

- kernel can see your thread! MP-aware
  ![[Pasted image 20211006120823.png]]
- 可以打散到不同的 cpu 去執行了
- 不再有 blocking code 的問題了
- Example :
  - Linux Pthread
  - Windows NT/XP/2000
  - Solaris 9 and later

### Many to Many Model

![[Pasted image 20211006121025.png]]
kernel 看得到所有的 thread 但是他不想要創一堆來一對一

- process contention scope 名詞後來不定會用到，指的是上半部的排程
- systme contention scope 指的是 kernel thread 排程

現在趨勢要馬就是多對一要馬就一對一，不會再在意去節省那一點點

### pthread

compile 編譯要下 pthread 選項 `-lpthread`

```c
#include <pthread.h>
#include <stdio.h>

int sum;
void *runner(void *param); //the thread

int main(){

}

void *runner(void *param)
{
	int i, upper = atoi(param);


}


```
