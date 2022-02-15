# Process Scheduling

## Scheduling Criteria

- CPU utilization : 希望可以靜可能的讓 cpu 忙碌，越多越好
- throughout e.g. 10s 完成 100 process，越多越好
- turnaround time : `執行時間 exec 加上 waiting `，越短越好，因為 exec time 基本上是不變的等很久表示有很多時間在等
- waiting time : process waiting in ready queue
- response 這邊定義的比較特殊，工作產生 `第一個 response`(跟 user 互動)的時間

### Scheduling Algorithms

越複雜的東西越不容易成功，因此這邊看到的方式通常都很簡單，重點是方法的性質分析

#### First Come First-Served Scheduling 先來先做

Process Burst Time : P1 24s P2 3 P3 3

|              | P1 : 24 | P2 : 3 | P3 : 3 |
| ------------ | ------- | ------ | ------ |
| Waiting Time | 0       | 24     | 27     |

Avg : 17

|              | P2 : 3 | P3 : 3 | P1 : 24 |
| ------------ | ------ | ------ | ------- |
| Waiting Time | 0      | 3      | 6       |

Avg : 3

啟發 :

- through put 跟 waiting time 沒有絕對關係
- 先來先做的話，一個超大的先來就慘了
- 對 IO bound 的程式很差，比方說我有兩個短的要 IO 他卡在後面不能先丟出去` I/O utilization` 可能會差
- 好處是 predictable 可以預測他的行為！ 在 computer science 裡面是個重要的行為，有時候會寧願犧牲效能但是可以保證在 xxx 秒內可以做完

- response time / waiting time 可能會很久
- CPU utilization `沒有問題`！就算就一直在等一個 IO 密集的的程式，只要有人在運行 CPU 就是在忙

#### 短的先做 Shortest Job First

![[Pasted image 20211013104349.png]]

SJF 是最強的，如果`已知`有哪些工作，沒有其他演算法可以贏過他
很重要，需要自己用數學證明 :

```
For each dispatched process, waiting time =
1st: 0
2nd: x1
3rd: x1 + x2
4th: x1 + x2 + x3
...
nth: x1 + x2 + ... + xn-1

=> Total waiting time
= (n-1) x1 + (n-2) x2 + ... + (2) xn-2 + (1) xn-1 + (0) xn

Obviously this sum is minimized if the xi's that are multiplied the most times are the smallest ones, i.e., x1 < x2 < ... < xn-1 < xn.
Thus, in non-preemptive scheduling, "Shortest Job First" (actually Shortest Next CPU burst) is optimal for the purpose of minimizing Average Waiting Time.
```

##### Non-Preemptive SJF:

- 不會停下來給短的做
  ![[Pasted image 20211013104907.png]]

##### Preemptive SJF :

做到一半如果來一個很短的，會停下來給他先做，注意是在比`剩下的時間`

Questions : Which one is the characteristic of SJF

- short average waiting time : 是的 沒錯剛剛有證明
- low waiting time variation : 肯定不是，因為短的優先所以快的就很快做完了，慢的就真的很慢，如果是剛剛先來先做很多人都會被拖到時間差不多

### Fairness v.s. Efficiency

- SJF 如果短的一直來，或者是有一籮筐短的，長的人等到死 starvation
- SJF is not a fair scheduling algorithm

  - 相對來說 FCFS 中 waiting time is always bounded 可預期

- There is a dilemma : efficiency v.s. fairness

### Determining Length of Next CPU Burst

shortest job first 很棒，但是在作業系統中不存在，原因是因為我不知道大家真的要跑多久而且會有 starvation 問題

### Priority Scheduling

- A priority number (integer) is associated with each process 越小越重要
- 這個 priority 要怎麼給是一個 policy 的問題
- SJF 就是一種 priority scheduling 他的 priority 是 cpu burst time
- Problem 是 starvation 問題
- solution : aging 過了一陣子都沒得到 cpu 我就把你的 priority 調高

### Round Robin (RR)

- 每個 process 得到一個小小的單位時間 (time slice / time quantum), usually 10-100 milliseconds
- 可以把它想成是一個先來先做得變形
- 為了 response 好
- 單位時間 q 跟 context switch time 比起來不能太小
- q 無限大的話其實就是 FCFS
  ![[Pasted image 20211013112851.png]]

![[Pasted image 20211013113336.png]]
time quantum 小的話，切的細分得均勻 better interactive response
大的話， turnaround 時間短

### RR vs FCFS

- RR response 比較好 (我想是因為 cpu 不會被某個 process 佔據)，一個 time sharing 的系統以 definition 來說就是會用 Run robin
- FCFS 的 turnaround time 比較好，可以考慮下面的栗子
  ![[Pasted image 20211013113652.png]]

### Multilevel Queue Scheduling

![[Pasted image 20211013113835.png]]
Queue 有 priority，一個 Q 做到`沒東西了`再到下一層

### Multilevel `Feedback` Queue

要先定義 :

- 多少 queue?
- 每個 queue 的排程演算法 scheduling algorithm
- 什麼時候要 promote process
- 根據什麼規則 demote process

#### Example Multilevel `Feedback` Queue

##### e Three Queues :

- Q0 : 最重要 RR with 8 milliseconds quantum
- Q1 : RR with 16 milliseconds quantum
- Q2 : FCFS

##### Scheduling :

- New job enters queue Q0 is served FCFS (RR), if it does not `finish` (沒有讓出 CPU 還在運算) in 8 milliseconds job is moved to queue Q1
- At Q1 if it does not `complete` (跟剛剛 finish 意思一樣) in 16 milliseconds demote to Q2

- `IO bound process` 那些 interactive process 跟 user 互動的 process ，我們想要 favor IO bound for

  1.  interactive response
  2.  IO uitilization

  - ，如果在 8 millisecond / 16 耗盡前就讓出來，我就把你 promote ，反正你每次只要一點點時間

- CPU bound process 要跑很久的，user 本來就預期他在那邊跑，因此往下 demote 也沒關係，而下層`時間切片大`讓整體 `turnaround time 小` & `throughput 提升`

老師說系統重點不在方法，而是背後的原理

### Solaris Scheduling

![[Pasted image 20211013115800.png]]

- priority 越高， time quantum 越小，所以很難留在這邊
- time quantum expired : 本來優先級是 59 你超時了我就把你丟到 49
- retrun from sleep : 表示上次是自願放棄 cpu 跑去 sleep 的，我就把你升級

## Real Time Scheduling

什麼叫做 real time? IEEE 定義要確保正確率而且，要確保一個 deadline 時間

soft real-time : 作業遲交還可以打個折，沒有保證一定！
head real-time : 工作一定要在 deadline 前完成

Real time Processes 有個特色 : `週期性`的獲取 CPU
因為通常是每隔幾秒去每秒去外界取個樣回來做計算因此通常， has process time t, deadline d, period p for 0 <= t <= d <= p
![[Pasted image 20211015154053.png]]

### Rate Montonic Scheduling

- priority 給他的週期的 inverse 頻率越短 = 短時間來很多的，priority 高 - 投影片是說幾個 milisecond p1 = 50, p2 = 100 因此 r1 = 1000/50 = 20 r2 =1000/100 =10 因此 r2 的 rate 比較高 - c1 = 20, c2 = 35
  ![[Pasted image 20211015154543.png]]
  p2 還沒做完 p1 又來了！所以還給 p1 做

#### Missed Deadlines with Rate Monotonic Scheduling

c 1 = 25, c2 = 35
p1 = 50, p2 = 80
![[Pasted image 20211015154734.png]]

p1 = 25/50 (每 50 秒來一個工作 要算 25 秒) 利用率 50%
p2 = 35/80 <= 50%
兩個都小於 50% 利用率，結果 deadline miss 了！

其實想一下蠻奇怪的，我有個工作快做完而且快 miss 掉了，而且有個短週期的殺進來就要交給他做，但實務上是採用這種而不是下面的 `EDF` 原因是因為 :

1. EDF introduces a larger runtime overhead than RM，因為 EDF 是最快到期的先做，因此如果有一個週期性的 `task` 他每一段時間丟出來的 job 我都要計算 job 丟出來的時間加上他的 deadline 時間 [^1]
2. 還有一個是他不 predictable 工作完成的時

### Earliest Deadline First Scheduling (EDF)

- 像是學生繳作業的行為
- Priorities are assigned according to deadlines

![[Pasted image 20211015155318.png]]

要求最小公倍數繼續往後排 (50, 80) 400

## Thread Scheduling

CPU 在做排程的時候看的是 kernel thread，未必看得到 user thread，除非用的是一對一

kernel thread 1 對１對應到 lightweight process

Unix 的話是拆成兩部分， process contention scope (PCS) System contention scope

pthread attr setscope(&attr)

### Linux Schedluing in version 2.6.23 +

### virtual clock

每個人發一個時鐘，理論上每個人時間都是一樣的，但是有人的時鐘可能走得比較慢精神時光屋的相反，那我就會多給他一些真實時間讓他跟上

#### Completely Fair Scheduler (CFS)

- favor 時間走得比較慢的

- nice

Priority 0 - 99 是 static 不會動的 real -time processes, 100-139 可以透過 nice 從 120 -20 or +19

時間走得慢的 virtual run time cpu 就會讓你 nice value 靠近 -20

## 多處理器排程

### partitioned scheduling

每個 CPU 都有一個自己的 ready queue，每個 process 就在各自的 queue 因此是獨立運作，linux 就是這樣做的，實務上這種方式較多

### global scheduling

process 全部在一個 queue 分給所有 cpu ， cpu 有 idle 就過去拿任務
好處 : 不容易有 cpu idle
壞處 : 實作比較複雜，常出現在分散式系統裡面

### Architecture 多處理器架構

#### SMP Symmetric multiprocessing

- cpu 用一個 shared bus 來存取記憶體，但彼此競爭 memory cycle / bandwidth
- 架構簡單、cpu 少的時候效能好
- 每個 cpu 去 access 變數 latency 都是相當的

![[Pasted image 20211020101701.png]]

### Non-Uniform Memory Access (NUMA)

- 這種架構 memory access time 取決於 memory 在哪裏，存取自己的就很快別人的較慢
- 擴展性好，連結多個 cpu 效能不下降
- 拓墣方式 : ring (intel), mesh, hypercube, ad-hoc
  - hypercube 特性，有 n 個點的話彼此最遠距離不會超過 lgn

![[Pasted image 20211020101945.png]]

### 多處理器排程要注意的點

以下兩者衝突！

#### Processor Affinity

- 有時候最好是不要移動，process 移動好像沒有成本，但其實現在 cpu cache 都很大 (L1/L2) ，我好不容易把常用的 cache 都建立出來現在搬過去又要重長 cache warm up， 一開始都會是 cache miss 降低 cpu 一開始的速度

#### load balancing

- 把 process 在 cpu 之間做移動來避免少數 cpu 在忙有人在閒
- Linux :
  - push migration : 把 process 從一個很忙的 cpu 推開
  - pull migration: 把一個 waiting process 從忙碌的 cpu 拉到閒置的 cpu
- 比較會發生在 patrioned scheduling 之間

## Multicore and Multithreading Processors

注意! : 這邊說的是硬體的 thread 跟軟體角度說的 process thread 不同，thread 指的是 `logical processor`邏輯處理器，emulate by `independent set of registers` (今天如果用了兩組暫存器，以軟體的角度就會覺得有兩顆 CPU 但其實這兩組暫存器共用一個物理 cpu 運算單元 ALU)

![[Pasted image 20211020104042.png]]
注意 : 此 thread 非彼 thread (in here is hardware thread)

- memory stall 是像是 lw, sw 仔入資料到暫存器
- compute cycle 是一些數字等等計算

![[Pasted image 20211020104449.png]]
muti-issue 指的是現在一顆 cpu 內部裏面有數個執行副本好幾個加法器、好幾個乘法器...但因為資料處理之間有時候會有相依性，產生很多 idle

這個時候如果有多個 hardware thread (多給他一個邏輯處理器) 就可以有效的利用這些加法器

### Hierarchical Scheduling Domains

![[Pasted image 20211020105108.png]]

- Logical CPU 就是一組獨立的邏輯處理器，其他都是共享的
- 這個時候 process 的擺法當然是一個 logical cpu 放一個，全部散開是最好的

[^1]: [Real-Time Systems](http://retis.sssup.it/~giorgio/paps/2005/rtsj05-rmedf.pdf)
