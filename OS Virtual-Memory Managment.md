# Virtual-Memory Managment 虛擬記憶體管理

[[OS Memory Management]]

不會馬上配置 physical 的 page，**真的踩到再給你**，有需求我再幫你處理 paging，你這個 page 沒有被用到我是不會幫你配置實體記憶體(frame)的

那我怎麼知道被踩到了?

```
	|     | frame | valid/ invalid  bit |
	| --- | ----- | ------------------- |
	| 0   | 4     | v                   |
	| 1   |       | i                   |
	| 2   | 6     | v                   |
	| 3   |       | i                   |
	| 4   |       | i                   |
```

要有一個 swap device (HDD 硬碟)，當踩到 invalid 產生一個 trap (page fault) 把裏面的資料讀出來放到記憶體 (所以我們也要記住資料在硬碟上的地址)

#### 有什麼好處?

- 每個 process 可以配比較少的 memory 資源，也因此建立比較快 fast response

#### 流程

不要忘記查找記憶體是 MMU 負責的

1. 一但 MMU 檢查 page table 檢查是否是個合法請求(看是否存在? 順道一提，之前看網路上講 malloc 只會在 page table 上建立好不會真的分配好 frame)
2. 如果上面 valid-invalid bit 是 0 ，他要負責幫我產生一個 **trap**
3. (page fault) 就是一個 interrupt，cpu 會到 IVT 中斷向量表查找，掉到作業系統 ISR 就會收到
4. 會檢查這個地址是否合法，如果合法的話去磁碟上查找他
5. 假設查找到了，ISR 會在**記憶體**找到一塊 free frame 把資料從**磁碟** load 上去 (issue a read from disk to a free frame)
   1. wait in queue for IO device (realse the CPU while waiting)
   2. wait IO finish job
   3. receive the interrupt from IO device
6. ISR 更新 page table (Correct the page table 更改為 valid)
7. back to user space and 等 CPU 重新把控制權給到這個 process
8. 重新開始中斷前的指令，再次讀取這個記憶體位址

以下最多發生幾次 page fault? 假設 instruciton & data 不會跨越 page boundary (page 大小通常是 4kb)

```
lw r1, r2, 12
```

2 次，instruction 一次，lw 取值一次

#### 要求地擊中率

計算之下發現為了讓速度只降低 10% miss rate 必須是 四十萬分之一
為什麼能達成呢？因為 swap 通常小小的 4GB 大約是 1:1

TLB 中

paging disk 10ms
4µs

#### TLB

cache 查詢 page table 的結果，比到 main memory 茶東西快很多
中了表示 TLB 裏面有 address mapping 資料，中了表示一定在 memory 裏面，不可能有 page fault
沒中則有

1. page fault
2. page hit TLB

## Page Replacement

雖然說 demand paging 用到才配置，慢慢執行慢慢執行越來越多的 page 被帶進來，如果不處理的話主記憶體終究會被塞爆，因此要把不常用的踢出去

### Basic page replacement

1. find desired page on disk
2. find a free frame
   1. if there's free one use it
   2. no free? find a victim

要取代一個 victim 要把他寫回 backing store(swap device )但是這段時間會 double page-fault handling
因此我們設置了一個 modify (dirty) bit 來紀錄有誰資料是有被修改過的，因為如果沒有被修改過我就不用寫回 swap device **(可是我怎麼知道她一定在 swap device?)**

### LRU

- LRU 有一個 stack property 指的是操作玩的結果 4 frame 會是 5 frame 的子集合，下圖，為什麼很重要呢 ? 這表示 LRU 一定是越多 frame 越好
- LINUX 使用 LRU
- 效果好跟最好的理想狀況相差不遠，為什麼呢？ 時間局部性（_Temporal Locality_）的存在，因此往前往後看現在在用往前往後看都會發現他有在使用
  便宜

#### LRU stack property

![[Pasted image 20211124115858.png | 400]]
給 n 個 frame 剛好是 n-1 的結果 + k，表示給比較多的 frame 表現只會更好，聽起來理所當然但不是每個演算法都如此

#### Page fault

![[Pasted image 20211126153254.png | 400]]
下降不是線性的，一開始下降很多直到一個 sweet spot ，跟 multi thread 現象一樣
原因是什麼 ? 因為最近常常使用的集合能被包括就可以達到 sweet spot，裝完之後再多裝的都是一些沒那麼常用的

![[Pasted image 20211126153912.png | 400]]
像是 FIFO! 老師用三跟四畫了一遍有點難體會，但主要就是剛好一直 access 剛剛刪掉的那個

老師說這是很微觀理論的性質，實務上造成的問題還好主要還是說 FIFO 就是廢

#### LRU 的簡化版本 Clock Algorithm

如果對一個 page 有操作就把他的 **reference bit** 設起來
複習一下 page 上的 bit 目前講到 dirty, valid/invalid, reference

![[Pasted image 20211126155252.png | 400]]
像是一個時鐘，外圍是 page 裡面一個指針一路走，如果是 1 我就把他設成 0 並且切到下一個，如果是 0 就把他回收掉，因此如果在兩次啟動的中間有重複使用那下次來又會發現變成 1 ，否則就維持 0 變成 victim，如果記憶體回收壓力大，指針就會越轉越快

#### dirty bit + reference bit

| dirty | reference |                                 |
| ----- | --------- | ------------------------------- |
| 0     | 0         | not modified, not used recently |
| 0     | 1         |                                 |
| 1     | 0         | 優先取代                        |
| 1     | 1         | better leave it there           |

老師的解答
要砍最近沒用到但是 dirty 的，不然看眼前的利益結果砍一個常用的砍掉了又回來又砍掉

我的想法
(0, 1) 最近有被用到 但 clean : 刪掉花 0, 0.9 的機率 花 1 = 0.9
(1, 0) 需要寫回但最近沒用到 : 刪掉花 1, 0.1 的機率 花費 1 = 1.1

### Counting Algorithm

- Recency : LRU / clock
- Frequency : LFU / MFU

LFU 的情況下，如果有個老人累積了一定的數量那他就會一直卡在裡面，因此實作會透過一些方式來解決，比方說 counter 數值 shift 一位半衰期，來讓那些曾經常用但後來不常用的慢慢死去

MFU : count 如果比較低

LRU 改進 :
https://www.itread01.com/content/1549095861.html 加上一個 hot / cold

#### LRU vs LFU (感覺就會考)

> 老師再次強調，做系統策略都是 keep it simple 簡單暴力就好，但是重點是分析

- LRU

  - high response to locality change 近期用的變了就可以跟著應變
  - sequential reference 會洗掉 (1, 2, ,3 4, 5, 6, 7, 8,9,10,11,12......k)
    - 老師說 sequential 通常是 one time 的，像是 streaming 不會拉回去看、資料庫搜尋就是搜尋過去、copy big file 也是一樣，因此不會想把他放到快取裡面，因此不會想把他們放到快取裡面但是 LRU 發現新的來就塞進快取
  - 直接做一個排序，不用額外記住一個 counter / time 之類的資訊

- LFU
  - Need cool down and warm up (累積足夠量以前都很危險，突然不用了也要等別人追上他的量或是等他半衰期掉下來)
  - 要保留一個 counter 多花一個空間

### Swap space Mangement

- 例外弄一個獨立的檔案系統，像是 linux
- 在原有的檔案系統切割一個 file: 像是 windows ，在根目錄底下有個隱藏檔案 C:\pagefilesys linux 也可以這樣用

去 scan swap map 找到沒有在使用的就可以放進去了，0 表示 free 1 表示用到 3 表示 有 3 個 process shared 這個 memory

## Demand paging 應用

### Page Sharing

#### Share code 程式碼共用

- gcc compile 我寫好的檔案有運用到 Standard c library ，但是其實沒有整份 copy 一份，整個電腦只共用一份 standard c library
- kernel code

圖片中
![[Pasted image 20211203153455.png | 400]]

#### Share memory

用 `shmget()` 做出一塊，然後用 `shmat?()`
![[Pasted image 20211203153737.png | 400]]

上面可以看到每個 process 都有自己的 page table ，這樣比較好實現共用
老師說相對來看 Inverted table 就比較難處理，他預設的架構變成是一蘭一蘭的 Frame 是被哪個 process 跟 pageid
老師問要怎麼修改才能讓他可以支援呢? IBM power 這顆就是用 inverted table (會考!)

### Copy-on-write

有說到 fork 出來的時候複製一份出來，最暴力的方式當然就是把原本 Process 的記憶體都扎扎實實複製出來，為了增進速度，會透過 page table 直接映射到原本對應的 Frame 上**並且標示成 read only**，因此真的有修改到的話 read only 會導致產生出一個 **trap** ，到 OS 這邊就會把他 copy 出來並標示成 read / write 接下來就可以對他做操作了，另外剛剛提到設定成 read only 以便偵測複製出來的部分有修改請求，這部分是要**透過硬體協助來偵測的** 有同學問為什麼不會被當成 segment fault，老師說就要看他 Segment table 是怎麼設定的，他會來檢查 Segment table 如果看起來都正常那就不會 trigger segment fault

![[Pasted image 20211203154152.png | 400]]

如果硬體沒有 MMU 就要改成用 `vfork()`

### Memory mapped files 很重要

我們可以把共用記憶體 attached 到 page table 上，同樣我們也可以把 file 映射到 Page table (作業系統是需要先從檔案系統把 file data 裝到 frame 上，再映射到 page table 邏輯上 high level 就覺得好像已經 mapped 到記憶體上了) 平常我們要透過 `read`, `write`... 現在就可以透過指針來操作檔案

Reading the memory segment triggers page fault (一開始 page 都還是 invalid, 讀取之後觸發 page fault，處理方式跟前面提到的一樣只是現在是從 file 拿檔案而不是 swap 拿 file )

寫的話就會產生 diry page，記憶體壓力大的或是平常有空的時候時候作業系統就會寫回檔案上，一班的檔案其實也是一樣只要有修改作業系統就會找時間把他寫回去

#### Memory mapped file 有什麼好處?

simple byte level operations : 修改比較快速，假設今天是一個稀疏矩陣
教授說 IO 是大檔案的時候比較有效率，因此比較有效率就是 1. 對資料的讀取比較精準 2. 不用每次花時間 system call 進到 Kernel

不用進 kernel (可以仔細思考)

### Mapping of pages

#### Anonymous page

原則上 code, data, stack 是屬於這類，被 Swap out 的時候會丟到 swap

## Performance Issue

### Thrashing 震盪

當一個 Process 分配到的 page 不夠，他就要 swap out 一個記憶體，結果又把別的 process 常用的踢掉(???) 因此 CPU utilization 變很低

![[Pasted image 20211203162220.png | 500]]
後面記憶體不夠大了，大部分的時間都花去 IO 上了

### Working set group

![[Pasted image 20211208101931.png | 400]]
老師說可以寄一下
在給定的 window 之下 (e.g. 1000 instruction) 所用到的 page reference
會隨著時間變化長大縮小，依照當下 tempral locality 強弱而定
![[Pasted image 20211208102004.png | 400]]
圖中的波鋒就是一段常用記憶體轉變的過程

### Trashing 發生了

1. 計算 total working set 就可以知道
2. 計算 Pf ratio (排除 working set migration)

- Swapper : 積極的記憶體回收
- Process termination : Linux 會用 Out-of-Memory(OOM) Killer as last resort (把 priority 低 memory size 很大的 process 直接殺掉)
  - Android 也有 LMK

### Background Dirty Page Flushing

要找 frame, 不夠了就要找 victim, 優先找 clean 的，因為想避免額外的 IO 還要寫回磁碟，因此時不時就會在 background 去把一些 dirty page 寫回

Linux : daemon

### Pre-paging ( Fre-fetch)

tempral locality(在時間上是群聚的，這個 page 被用到他等等也很可能被用到), spatial locality(這個 page 被用到，他的鄰居很有可能也被用到)

因為有 spatial locality, 當一個 page fault 發生時我可以把他的親戚一並載入兩個好處 :

1. disk IO 次數比較少 (一個 IO 5ms 可以做好多事情啊)
2. 減少 page fault 次數

![[Pasted image 20211208104342.png |500]]

可以發現以下兩個都會被影響到

1. cpu cache miss ratio
2. memory miss ratio

#### 陣列擺放位置

row major (C 語言)

```
0 0	0 1 0 2...
1 0 1 1 1 2....
2 0
3 0
4 0
...

00 - 128 0 第一個

```

column major (fortan)

```
0 0	1 0 2 0 3 0...
0 1 1 1 2 1 3 1...
0 2
0 3
0 4
..

```

You can test by creating a 2-row 3-column array and assuming it will be stored in row-major order, then checking. So create this:

```
0 1 2
3 4 5
```

Now look at the bytes in memory. If they go:

```
0 1 2 3 4 5
```

it is in row-major order. If they go:

```
0 3 1 4 2 5
```

your program uses column-major order.

Normally, C uses row-major ordering and fortran uses column-major ordering. I said it depends _"to some extent"_ because with Python, for example, you can specify either ordering and even mix them on a case-by-case basis in the same program.

### TLB Reach & Large Table

回顧一下我們都是先查 TLB 沒中再來查
TLB Reach = TLB Size x Page Size (因為一個 page 一個 TLB entry)

- TLB Size 加大 : 很昂貴不切實際
- Page Size 加大 :每次都被迫 allocate 很大一段
  - space locality 好才會好

### 實現不同 page size / trade off

在 linux 有個 `huge page`
實作起來其實也很簡單，就是 multi level page 不去查第二層
![[Pasted image 20211208110331.png]]

Large Table Size:

- page table 就小
- TLB reach 提升
- 但是有可能帶入很多 unused data into memory

### 有哪些 bit 什麼時候用到

老師說重要!!!
RW :
Dirty:

4kb 因此只需要 12 bit? 但其實 12 bit 不用使用??怎模去算的
![[Pasted image 20211208110917.png]]

考古有問 page table size，老師說最必要的是表達 frame number 是一定要的，後面的 valid, dirty....可以寫說我不考慮這些因此我只算了...或是我考慮這些

一個 32...4kbpage 那至少要
page table 的大小本身是很多個 page 去儲存的
