# Mass Storage System

### Magnetic Tape 磁帶

- sequential read-write 很快 (400MB/s)
- extremly slow on random access
- 磁帶機貴但卡夾很便宜
- 18 TB per cassette

### Magnetic Disks 機械式磁碟

前面有提到 disk IO 大概都是 10 ms 起跳，DRAM 100 ns，CPU cycle 1 / 4Gz

- Transfer rate 是機器到外面的水管 e.g. SATA 3 : 600 MB/s
- Positioning time (random-access time) Seek time (move the disk heads)+ Rotational Latency(disk to rotate the desired)
  - 5400 rpm ~ 7200 rpm => 5ms~7ms average seek time

一圈磁碟叫做一個 track 所有一樣半徑的 track 整個叫做一個 CYL cylinder
一圈 track 被分成一個一個 sector 通常 512 Byte
通常會用 C/H/S 哪一個 cylinder, 哪一面，哪一個 sector 但討論會把面去除簡化掉

#### 磁碟介面

- ATA/IDE
  - PATA, SATA (南橋 IO hub)
- PCIE (走的是北橋 memory hub!!)
- SCSI
- FC

#### Disk Scheduling

- Operation system is responsible for using hardware efficently
- Access time has two major components
  - Seek time : easier for OS to control
  - Rotational Latency : hard for outside software to 了解現在轉到哪了

##### The Need for Disk Scheduling

磁碟排程是一個怎麼樣的問題? Why we have lots of the disk IO??
如果只有一個 Process 而且只有讀的話，因為會讀完之後才會往下跑所以一次只有一個 IO
但

- Multiprogramming
- Write buffering (marked 成 dirty 就離開了，cache 送來的就是一大票 (**沒聽清楚什麼 cache**))

How to select the next request to serve? 老師說骨子裡是 TSP (旅行推銷員問題 Travelling salesman problem)
讀取的順序會很大的影響 response and throughput
![[Pasted image 20211222115421.png]]

## 磁碟排程

### 先來先做

很公平，不會 starvation 但效能沒那麼好

### 近的先做

效能好但也許會被綁架，一個很遠的就 starvation 了

### SCAN

類似電梯的一種方式，掃過來走到零再掃過去
![[Pasted image 20220110212431.png]]
打到底叫做 full stroke 因為走到底才會掉頭因此最差的時候會變成兩個 full stroke

### Selection of a disk-schedyling algo

Linux 是一種大鍋炒~

- starvation : FCFS
- seek time 最佳化 : 電梯法
- 會被檔案系統 file-allocation method 所影響存取效率
  - 之前提到 ext 系統把他分成幾個 cylinder ，metadata 跟實際資料不遠

### Disk Seek Optimization

- 磁碟排程是難的，NP-hard ，我們沒有要追求最佳解
- 磁碟讀取檔案還有一個讀取的部分 (rotation delay) 這部分 OS 很難做
  - 韌體本身比較知道碟片轉到幾度角......交給他排成就好
  - SATA NCQ(Native Command Queuing)

## RAID

### RAID 0

- Block striping
- 效能好 Best performance，東西交錯左右放，好處是可以平行的讀取
- 但可靠性低，N 顆磁碟一顆壞了就全毀了

### RAID 1

- Mirroring, 100% redundancy
- Less efficient
- More reliable

### RAID 2

老師說是廢物 XD

### RAID 3

老師說沒看過不講

### RIAD 4

- 四顆硬碟 放 A1, A2, A3, Ap, Ap = A1 xor A2 xor A3
- 因為 Ap xor A1 xor A3 = (A1xorA2xorA3)xor A1 xorA3 = A2
- 最簡單的 pairity 可以接受壞掉一顆
- 缺點是每次要更新資料都要同步更新 pairity，因此每一顆硬碟上要修改都要去搶 parity 那顆硬碟
  - 把 $D_1, D_p$ 讀出來
  - $D_p' = D_p \oplus D_1 \oplus D_1'$
  - Write back $D_p'$
  - Write back D1

### RAID 5

- 是個好東西!
- 改良 RAID 4，把 pairity 散放到各個硬碟上，算是蠻標準的做法，其實就是 RAID 0 + parity 保護
- 可以用硬體也可以用軟體來實現 (因為 xor 算得很快)

來算一下可靠性，假設一個硬碟一年內都正常運行的機率是 $p = 0.99$
RAID-0 1-year up : $p^4 = 0.96$
RAID-5 1-year up : $p^5+(5,1)(1-p)*p^4 = 0.99999$

可以兼顧平行度讀的效能以及比單顆硬碟更好的可靠性!

### RAID 6

- 有兩個 Parity stripe 因此可以與許壞兩顆硬碟
- 沒有辦法用軟體做，因為沒辦法用 xor 來做了，不用硬體會來不及
  - Reed-Solomon code
  - EVENODD

## SSD

不是真正的硬碟只是 **emulates** standard block devices
以前的 SSD 是 DRAM + 電池 (因為是揮發性的要避免斷電的情況) ，後來改成用快閃記憶體 flash 居多

為什麼可以盛行呢？ 因為設計上採用最小侵入式的方式，用一樣的溝通介面 ＳＡＴＡ 作業系統其實也不知道他接上的是一顆 SSD

Embeddde Flash card (手機有 UFS EMMC 兩種介面)、SD cards、USB thumb drive、SSDs、PCI-e, flash cards

RAM SSD > flash SSD >> HDD

### 應用

- Cloud storage : tier storage 可以做一層一層有種層層快取的感覺, cache SSDs
- Personal computer : 取代 HDD
- Embedded device : 手機、智慧手錶

![[Pasted image 20211229103044.png |500]]

- 中間的主控 main controller (二到四核心, 400MHz-800MHz, 32bit) 是一個蠻強的控制器，已經差不多是幾年前智慧手機的規格了
- 旁邊有一個 DRAM
- 一個 flash array 圖中有八個?背面通常還有

**要特別要注意的是** :

1. flash physical characteristic
2. firmware design
   通常的確插上去電腦就可以用了，所有管理的都透過硬碟上的 firmware 來做了，但一些情況還是會回到 OS driver 來處理

#### flash physical characterristic

兩家的硬碟效能可能會差很多，原因是在演算法的差異

![[Pasted image 20211229103843.png | 600]]

橫軸是密度，縱軸式速度快慢

- 最快的是 SRAM 用在 Cache 上面 (register 用的是[正反器](http://yhhuang1966.blogspot.com/2019/06/latch-flip-flop.html))
- 儲存用的是 NAND flash memory 容量很大但比較慢
- PRAM 老師說位置有點畫不對，容量應該比 DRAM 大，是相變記憶體主要應用於 IO Cache 加速用 (intel optane DC NVD??) 企業用

### NAND Flash

![[Pasted image 20211229104652.png]]
floating gate 交大施敏教授提出的，老師說不要有抗拒排斥的心態，身為資工人這些都必須懂才能做出有效的演算法來管理

- binary state，tunnel oxide 是絕緣體充電進去電子就在裡面了，左邊把電荷都困在裡面右邊把電荷都趕出來
- 電子在絕緣體中衝來衝去是因為電子在絕緣層衝來衝去會破壞結構
- 上圖一組就是一個 cell

![[Pasted image 20211229105245.png]]

- 左邊 SLC singel level cell 只有高電位跟低電位
- 但我們希望在不增加電路複雜度的情況下增加容量，(注意圖中的四種店為一次只會在一種) 因為有四種電位我就可以儲存 2 bit 的資料
- SLC 較為耐用、讀取差不多、寫比較快、貴 （軍用！市面買不到)
- MLC 反之，因為他分得很細因此壽命減少 33 倍、寫入也比較慢、 cheap!
- TLC、QLC 現在買得到的是這種 3, 4 個 bit

### 操作 NAND Flash

![[Pasted image 20211229105935.png]]
一個 block 2MB 左右，一個 page 通常 16kb
讀寫是以 page 為單位發生的我可以單獨跟快閃記憶體說我要讀這個 page
我們剛剛有看到一個 erase 操作，他是以 block 為單位做執行的，這給管理帶來很多的挑戰
![[Pasted image 20211229110222.png]]

- 小小的是 cell
- 一橫排是 page
- 整個圖上是一個 block

誒！我本來以爲 page 是一個虛擬的概念

### FTL (Flash Translaiton Layer)

- 位址轉換 logical to physical translation
- garbage collection
- wear leveling

#### logical to physical translation

![[Pasted image 20211229112128.png|300]]

- why? pages can't be overwritten unless being erasd
  - 因為物理特性，跟 DRAM 磁碟不一樣，**不能在同一個地方直接重複寫入**
  - 要直接原地覆寫一個 page 就要去檢查附近同 block 鄰居還有誰是有用的把它移到其他地方，並且整個 block 擦除
  - **out-of-place update** (老師說之前就講過了 [[OS File System#Log-Structured File System LFS]])
  - 因此需要一個 table 去記住他最新的位址 from logic sector -> physical page number
    - 老師說 page table 用到的技巧都可以用上 (Multilevel page table, hash...)
  - 這個 mapping table 通常會放在磁碟的 DRAM 上加速查詢

#### garbage colleciton

![[Pasted image 20211229112510.png]]
隨著時間推移，到處都是舊版的檔案 A, B 都是 garbage，我又不能直接整個 block 擦除

- copy 整個 block 有用的檔案到 reserve 然後擦除?
  - 如果整個 block 只有一個垃圾這樣忙半天只回收到一個 (複製 8 回收到 1)
  - 如果能找到都是垃圾的或是佔大多數的會很棒

#### wear leveling

- 一直充放電絕緣層死去
- 一個 block 通常可以忍受 3000 擦許循環

![[Pasted image 20211229113556.png]]
但是因為寫入的 temperal locality

- file system meta data 就會反覆的一直被更新
- 有些東西(影片檔案)就基本上只會讀取
- 因此就會有些 block erase 次數特別高，有些根本就沒有被 erase
- ![[Pasted image 20211229113825.png]]
  - 就會在兩塊藍色之間擦來擦去
  - 某些 block 早早就操勞過度導致整顆硬碟早早死去

要打破上述的僵局就要做資料的搬移，比方說把某一塊 Read Only 跟藍色交換
