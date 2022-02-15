# Memory Management

- 額外補充恐龍書
- cpu 所生成的地址稱作邏輯地址跟我想的一樣在在邏輯地址與物理地址不同時，邏輯地址又被稱作虛擬地址 virtual address
- 別忘了地址轉換這件事是 ＭＭ U 在負責的，底下有講到詳細流程
  - CPU -> MMU -> Memory

### Memory Hierarchy

register->cache->main memory->electronic disk -> magnetic disk....
Caching 就是把資料移到更上層更快的記憶體

![[Pasted image 20211110114710.png]]
老師說對特定的數字要有 feeling 不然討論會失焦

- 磁碟的 access time latency 是 5 ms 一秒鐘可以做 200 次 (而已) 的意思，當然一次操作可以搬移很多資料
- main memory 是 100ns 跟磁碟就差了 10^6 倍
- cpu cache 又快了 10 倍
- register 又快 10 倍

### Address Binding

程式會有 instruction (一條一條的組合語言) 跟 data (變數)，他們都需要被賦予記憶體位置，不然不知道要 jump 去哪裏，要 load/store 去哪裏

而記憶體位置的決定時機

#### Compile Time 決定

- Absolute addressing ，目前沒人這樣用了
  compile 的時候都寫死了， 100h 開始到 300h Store 100H, R1, jmp 234
  這樣表示這個程式一定要載入到他所寫的記憶體位置，如果有另一個 program 也這樣寫就彼此衝突了

#### Load Time 決定

程式要執行 `./a.out` 的時候才決定載入到哪裡，產生的程式碼必須是 `relocatable code` 也就是程式碼裡面的取址必須是`相對定址`

```asm
cmp ax, 0x1				0 : 66 83 f8 01
je  0 <_main>			4 : 74 fa
```

上面的 `74` 是 opcode `fa`是`偏移值` in hex (-6) 因此不管我載入到哪裡他用相對位置跳都不會有問題

#### Execution Time

可能執行的時候才會 binding 一些程式碼，甚至整個程式碼可以 migrate 到另外一段記憶體位址

- 需要硬體 MMU 的幫忙

因為我用的指令都是相對於 base ，當 CPU 發出 `logical address` 346 `MMU` 會幫我加上一個數直來換算成實際的物理記憶體位址

#### MMU

![[Pasted image 20211110120318.png |500]]
CPU 裡面的一塊 componenet

- relocation
- paging 虛擬
  - address mapping
  - virtual memory
- Segmentation 保護
  - memory protection

## Dynamic memory location

### 總結比較

#### Buddy System

- 找不到合適的就二分往下切，清空的時候如果鄰居也空了就合併
- has little external fragmentation why? 因為
- internal frag 會有，因為如果比小的大一點點又比大的小很多

#### First Fit

- Internal fragmentation 很多，因為他有可能找到的第一個空格比他需要的大超級多
- external frag 看情況

#### Best Fit

- internal fragmentation 最佳，因為他會找一個跟他嘟嘟好的
- external frag 會有，因為常常想找差不多大小的洞，放進去之後剩下一咪咪

#### Worst Fit

- it tends to minimize the effects of external fragmentation? 因為放進去都會剩很多?

### 新 Process 要塞去哪裡

設定一個 rellocation register

### Heap Mange in LINUX

記憶體不斷的配置，原本的地方塞不下了 malloc 會呼叫 brk()/sbrk() 讓 data 區段往上長
但跟前面 process

### 問題本質

![[Pasted image 20211112154134.png | 300]]
這邊就是一個 policy 的問題，如果有一些 free hole, 要放哪裡 ? 現在實務上是用 best / first fit

- Best fit : 找一個`最接近需求`的空間，比較可以留下連續大段空間，但可能找很久
- Worst fit : 找夠大裡面`最大的`，理想是希望可以剩很多空間讓我們繼續運用
- First fit : 從上往下找，第一個合適的就放，好處就是快
  實際模擬是 best fit / first fit 比較好
- buddy system?

### Fragmentation

#### Internal Fragmentation

給得比我要的還多，造成浪費
Best fit, First Fit 基本上不會有這樣的問題?，buddy system 才會像是我要 30 他開了一個 32 的空間來放

#### External Fragmentation

放了放拿了拿變成斷斷續續很多破碎的空間，buddy 不會有這樣的問題因為他最小就是切割到一定的量而且隔壁鄰居
我的 free space 是破碎不連續的，全部加起來有 80 但就是放不下
我自己判斷 best 最嚴重 再來是 first fit 最後是 buddy

### Buddy System

- 二拆法運行起來相當快，配置時間相當快
- 但是 total fragmentation 較大
- 把空間不斷的二分，直到差不多符合你需求為止，比較特別是釋放的時候，不是 buddy 的部分不會結合，只有在 buddy free 的情況會結合再一起
  可以用 binary tree 的方式做

### Case study

GNU c library malloc() 使用的是 best fit 的一種變形
Linux kernel 用的是 buddy system 還有 slab allocator 因為作業系統希望速度快點

### Slab Allocator

External fragmentation 的主因是因為配置的大小不一，因此前人放的位置 free 了之後下一個人不能用
極端點來說，如果每個人需要的空間都一樣大肯定不會有任何的 external fragmentation

kernel 裏面需要配置的結構主要就是那幾種 small object, 而且配置、deallocated 的非常頻繁，因此就用一個 cache 都配置到同一個 pull

### Solution

#### Compaction 緊縮

重新搬動調整記憶體內容把空間空出來，但就要花很多時間

#### 允許物理記憶體位址是不連續的

允許物理記憶體位址是不連續的，這樣 external 破碎一堆沒關係，拼一拼繼續用，要實現這種方式有兩種互補的技術 Paging & Segmentation

## Paging

- compaction : 前面提到會留下一些洞，external fraction，要解決這個問題可以透過 compaction 把資料搬動整理讓他們連續，簡單但是 overhead 大因為 process 要停下來等
- mapping approach ? : 是不是可以透過虛擬記憶體 mapping 到實體 ? 實體不連續沒關係，虛擬連續就好

- **frame** 是物理記憶體 **page** 是虛擬記憶體的單位，兩者大小一樣大概 4kb 一一對應 (根據經驗法則)

![[Pasted image 20211117102546.png | 400]]

![[Pasted image 20211117102802.png | 400]]
cpu 讀取 instruction 丟出一個地址，會被打成兩段，page 跟 displacement (在一個 4kb page 裏面再做定址 4kb = 2^12 12bit) displacement 不用轉換，只要透過 **page table** 查找 page 對應到的 frame

![[Pasted image 20211117103535.png]]

1. 這邊有 4 個 page 所以 0-3 =>2 bit
2. 8 個 frame =>3bit
3. displacement 0-3 =>2 bit
4. logical address => 4 bit
5. physical => 5 bit
6. page table entry => 我猜是 3 個

![[Pasted image 20211117103749.png]]
free frame list 順序是亂亂的，因為有些 process 來借來還，沒關係就抓幾個給新來的 process ， process 看起來是連續的但對應出去

### Paging hardware with TLB

**重點 :**

- page table 在哪裏 ? 在主記憶體裡面，因為她太大了
- 一個 process 一個 table，每個 process 看到都是 0 到最大 page-table base register (PTBR) page table length (PRLR) 他們屬於 process context 的一部分
- 因為 page table 在主記憶體裡面 cpu 要取址拿東西，(main memo 查找 100ns)現在變成要查找兩次了變成兩倍慢，平常 cpu 運行速度都是被記憶體拖慢 (lw sw) 因此可以暫存最近的放在 TLBs

- TLBs 是 fully associate memory，以下的流程都是 hardware 做掉的，作業系統的角色在 set up page table

![[Pasted image 20211117104644.png]]

[網路關於快取好文](https://hackmd.io/@drwQtdGASN2n-vt_4poKnw/H1U6NgK3Z?type=view)

### TLB

#### TLB hit vs Locality of reference

TLB 基本上沒辦法做大，通常是 128 個 entry 左右，這麼少會不會常常 TLB miss?
老師說這張圖要好好感受，做作業系統都會常常提到 **locality**
memory foot print ，一個小黑點代表一個 memory

![[Pasted image 20211117105618.png|400]]

- temporal locality : 現在用到這個 page 等等很大機率還是會用到，時間上的群聚性
- spacital locality : 一個記憶體位址被用到，他附近的位址也有比較大的機率被用到，sequential 的 access 微觀來看就是會擊中某一個 Page

### Structure of page table 設計方案

#### Hierachical / Multi Page Table

為什麼要有 Multi level Page table?
算一下假設 32 bit 系統 4 kb ，記憶體位址 2^32 / page size 2^12 => 需要 2^20 個，假設一個 4 byte => 4 MB 這樣有 20 個 process 就要 80MB

80386

- page table 可以被打散，這樣也可以減少找放他空間的壓力 ( 4MB)
- page table 一開始不要給全，有用到再來配置再來長出來

![[Pasted image 20211222224917.png]]
壞處是什麼? TLB miss 的時候要查兩層，

```
[i][j][offset]
10bits 10bits 12bits
```

第一層 : outer page table / page directory ， i 個 entry

第二層 : page table j 個 entry

老師說基本上這就是一個 `radix tree`

第二層存的直覺就是 frame number 而第一層存的指標會是物理位址，因為又用邏輯位址的話會反覆 recursive 的查找，worst case latency 更大

#### Inverted Page Table

在反向頁表（inverted page table）中，可以看到頁表世界中更極端的空間節省

![[Pasted image 20211119153339.png |600]]

基本上這個 table 也是一個 hash table，實作還是 page -> frame 但是重點
管理紀錄的是 frame 到底被誰用到因此整個機器只有一張，傳統的是紀錄 page 用的是哪個 frame

現在，要找到正確的項，就是要搜索這個數據結構。線性掃描是昂貴的，因此通常在此基礎結構上建立 hash table

#### Hash Page Table

改用一個 hash table
forward : commly implemented using multi-level tables, not hash tables,
inverted : 剛剛提到的，不用 hash 就要變成用線性了

下面圖表示一個 hash table 後面接著一串 link list，比對查到第二個才找到
![[Pasted image 20211119153240.png]]

## Segmentation

![[Pasted image 20211119154124.png]]

- 需要 MMU 支援
- 紀錄 segment 的用途是什麼？ 需要的 access permission

- RAM 裡面會有一個 segment table
- 我的 process

假設現在是 segment 3, displacement x 3200+x < 3200+1100 看起來是防止不同 segment 不要互相踩到
另一個是 **segment protection**

- 恐龍書提到 : 可以透過硬體的方式來做到 segment protection，透過 cpu 去比較 "每個 process base address 加上他最大可以到的 limit register 以及現在所要訪問的**邏輯地址**"

### segment protection

- text : 可讀不可寫可執行
- stack : 可讀可寫，拿來放我們的 activation record (local var, return value....) `可執行？？`

#### Stack Overrun Attack

![[Pasted image 20211119154940.png |500]]
假設現在運行 strcpy() 我讓你複製到一些惡意程式碼，然後最後的返回位址寫一個我在 stack 上寫入惡意程式碼開頭的地方

c literal "hello world" 應該是擺在不可以改的地方
老師說 gcc 有點偷懶把 literal 放在 code 尾巴

```c
#include <stdlib.h>
#include <stdio.h>
int main(){
	char *p = "helloworld";
	p[0]='H';
}
```

#### Paging and Segmentation

- paging 的動機是來解決 **fragmentation probelm** at process level
- segmentation provides memory protection

老師說現代處理器兩個都會支援，差別只是先 paging 再 segmentation 還是反過來

### example : intel 80386+

selector offset 去查找 segment table 找到 segment discriptor 加上 offset 做一些檢查

剛剛那個是 386 硬體提供的機制，那以軟體來說?

#### Linux Segmentation on Intel 80386

segmentation page 其實提供的服務有點重複

- read write protection -> both
- privilege -> both
- executable -> segmentation only

RISC architectures often have limited support for segentation

因此 Linux 沒有必要不會用 segment 能用 paging 就用 paging

6 global segments

- Kernel code
- Kernel data
- User code
- User data

uses 2 protection levels ?? 教授說現在不確定了 (80386 has 4)

### Reference

[[18.pdf]]
[[19.pdf]]
[[20.pdf]] 提到反向頁表等等
