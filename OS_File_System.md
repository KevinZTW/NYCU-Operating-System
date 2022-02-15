# File System

[[42.pdf]]

### Layer File System

![[Pasted image 20220110211949.png |500]]

```
	fread()/fwrite()	 Applications
	---------------------------------			User Space
	fs->read, fs->write  File-System interface

						File-System
						Implementation

	Read write to		Page Cache    Ch9
	page cache

						IO Scheduler Ch12

	Block read
	Block writes		Disk driver  Ch12

	Control signals		Disk

```

### Linux VFS

在 Linux 中有一層 VFS 虛擬介面來抽象化，在底層的檔案系統跟他註冊
作為一個軀殼在實作上定義了很多的 物件 object 在不同的檔案系統在磁碟上有他自己的定義不一定可能向ＢＴＲＥＥ或是一個表格，檔案系統要負責在他定義的檔案系統跟 Object 之間做一個翻譯，VFS

### In-memory kernel objects of linux VFS

- Superblock
  - one per FS, representing the entire filesystem
  - FS type,
- Inode
  - 每個檔案獨有的
- File object
  - representing an opened file. one for each fopen instance
  - 假設有兩個 process p1, p2 都對同一個檔案呼叫了 fopen 他們讀寫的位置可能都不一樣
- Dentry object
  - 有點類似樹的資料結構用來表示目錄

META DATA
每個檔案系統都設計得不太一樣有其獨特之處，是檔案系統設計的關鍵
LINUX EXT : 前身是 unix fast file system, 後來 ext -> ext2->3->4 , 是很具指標性的檔案系統
微軟 fat file system

![[Pasted image 20211215103627.png]]

可用空間表示 bit map 代表的是後面的儲存狀況 block 是否被使用了
有趣的是他把 inode 幾乎是照抄的儲存在磁碟上，歷史原因是因為這個檔案系統跟 keRNEL 是一起設計的

一定要問為什麼，為什麼要這樣設計不然皮毛也沒學到，這個系統是很具有巧思的

FAT 是微軟的，現在主力是 NTFS ，fat 不太在乎效能就簡單所以常用在隨身碟，簡單堪用但以裝置數是使用最多的，要使用還要付微軟錢

![[Pasted image 20211215104141.png]]

dir 對應到一個
左邊的指標連來連去，對應到右邊的 block

## Key Issues of Files System

- Directory implementation 要能夠展示、索引一層層的檔案結構
- Allocation (index) methods 我要讀檔案的某個地方，我要能去磁碟上找到他存的位置 (index)
- Free-space management 我怎麼知道哪些地方是可以用的

how to delete file? 其實在 fat 他只是把他標註為沒有用，就結束了
![[Pasted image 20211215104905.png]]

### Issue1 Directory Implementation

- linear list : 一個找一個找下去
  - 簡單的設計但是比較慢，FAT 為代表 Ext2
- B-trees (or variants)
  - Efficient search
  - XFS, NTFS, ext4(H-tree, fixed 2 levels)
  - scaling well for large

### Issue2 Allocation/Index Methods

怎麼從磁碟上去分出我所需要的空間? 怎麼去找出檔案在磁碟上的位置?

#### 連續配置 contiguous allocation of disk space

跟 IPv6 header 有點像，紀錄他的起始位置跟他的長度

![[Pasted image 20211215105702.png |300]]

- 所以今天檔案來的時候我會在磁碟上找到一塊物理連續的空間來把他放進去
- 好處，很重要! : **在檔案讀寫上是最完美的** Perfect for IO overhead，在檔案上讀寫 sequential 的話在 IO 上也是，之前有提到 sequential 的讀寫是最有效率的
- 壞處 :
  - 當檔案想要長大超過我給他的空間就很難處理，留大給他也許用不到、留小可能不夠，如果要搬移也需要成本
  - 浪費磁碟上的空間

#### Link Allocation

![[Pasted image 20211215111420.png| 300]]

規劃成一個一個 block，這些檔案散落在磁碟的各處，透過 block 開頭的指針來做串連，但是每個 block 開頭都有一個 pointer ，計算上變得麻煩因此又可以把它拆出來

- 優點就是檔案非常容易長大，有空的洞還是可以利用而且設計簡單
- 壞處是 random access，檔案破碎讀寫變成 **好幾次 IO** 要跟 disk 說喔我要讀 3000 開始五個 然後 8000 兩個， **還要去走 linked list**，在檔案中定位也要去走那個 link list

FAT 會有一個 Bad block list 來把壞軌串起來
file 也會串接一個接一個

空的地方就會標示為 0 因此要找空的地方要掃一輪為什麼是這樣設計呢? (paing free 的地方是用 link list 串接起來)

#### Indexed Allocation

![[Pasted image 20211215112639.png | 300]]

集中 pointers 到 index block
好處 : 我要讀檔案中 offset 10 我可以很快換算找到對應的 pointer ，如果是在上面 link allocation 要一個接一個的找才能走到第 10 個，有點 sequential 的味道了

- **但是檔案大小受限於 block 大小**

假設一個 block 512 Byte, ptr 佔 1 Byte，一層最多可以支援多少 ? index 512 x 512 = 262144 byte 顯然不夠，那就加到兩層

- **因此可以改成用兩層式的**
  1st level : pointers to index tables
  2st level : pointer to data blocks

#### 實際上 Inode 設計

![[Pasted image 20211215113643.png | 500]]

有點混合式的，前面有一些 direct block(pointer) 會直接指向檔案，後面有 single, double, triple indirect
為什麼要這樣設計，是拼裝車嗎?

![[Pasted image 20211215113822.png | 500]]

**老師說這邊是可以出計算題的，給我 single, triple 大小多大....**

可以看到大多數的 file 在 2K 左右都小小的，這些很小的 file 我不需要多層的 index 可以先用 inderict 來解決

![[Pasted image 20211215114320.png]]

可以看到這邊的概念跟 paging 很像，學系統就是這樣，很多概念是互通的

#### Extent- Based System

- **Linux ext4 為此類型**

- 大小不一， Extent 照字面翻就是一個範圍的意思，想要改善連續配置的問題的混合解法 (contiguous allocation + linked/indexed allocation) 現代檔案系統基本上都是這種

- 因為我還是希望在物理空間上是連續的，因此我可以當下`預測`檔案長大的可能性找一塊對應的連續空間給他 (可能依照副檔名) 長太大再串連起來

### Issue3 可用空間管理

#### Bit Vector (Bit Map ?)

```
1001   用了 沒用 沒用 用了
1111   用了 用了 用了 用了
0000
0101
```

- Bit mpa 需要額外的空間
  - block size = 2^12 bytes
- **這點特別重要** : 很好找連續的空間，因為我只要看有沒沒有一個 DWORD / WORD 是不是 0，是的話就表示 32 bit / 16bit 都是 0?
  - 連續這麼重要? 因為檔案配置連續 IO 才能有效率

#### Linked Free Space List

- 很快就能找到空間 (allocating and deaalocating free blocks in constant time) 因為空的空間是串再一起的，我只要看頭就知道大小， bit map 還要一個一個掃
  - FAT free block 是 0，配置了才會 linked list 串起來，可以感受到也有那種味道他想要配出連續的位置空間
    ![[Pasted image 20211217154424.png]]
- 不會浪費空間，之前有說過，破碎的空間還是可以串起來
- 但是不連續，效能差，磁頭移動成本增加，現代系統幾乎沒人使用

### File Fragmentation 檔案老化(破碎)

當舊的檔案移除後變成有一個一個的洞，把他們接起來拿來用的話就會變成讀取的時候要到處找，隨著時間過去洞會用來越多，越難找到連續的空間，各種系統都會有這種現象只是程度的嚴重而已

- Degree of Fragmentation (DoF) of a file

$DOF = \dfrac{\# \ of\  extents\ of\ the\ file}{\ the\ ideal\ \#\  of \ extents\ for\ the\ file}$

e.g. F2 100k ，在磁碟上找了三個空間 = 3 extent, DOF = 3 / 1 = 3

file system 越滿的話比較會發生，Degragmentation 偶爾可以做一做

## 效能

每個 file systme 都可以有一些獨門絕技，重點是能**說得出來他為什麼這樣做** 老師強調!

ext4 用了一缸子優化技巧
![[Pasted image 20211217155714.png]]

- 他把 partition 分割成一區一區的，每一區的開頭都有 meta data 因此 meta / user data 靠的比較近，磁頭移動比較少，可以看到下面的 fast 靠的就比較遠

![[Pasted image 20211217155800.png]]

- 回到這張圖可以看到很多檔案其實都非常小，甚至有很多 size 是 0 的檔案 (應用程式生成用以作為其標) 這樣正常讀寫 directory 有個 file1 entry ->file1 inode -> data block IO 三次磁頭跑來跑去，檔案很小的時候我乾脆把他 embedded 在 directory 底下，inode, data block 都不要了

- ext4 會盡量猜你會不會長大，讓檔案盡可能連續

![[Pasted image 20211217155917.png]]

#### 其他 OS :

- 磁碟快取 (tempralocaltiy) 讀過的資料還會再重複使用，就用 virtual memory page cache 來把他快取起來

![[Pasted image 20211217160714.png]]
被讀入的檔案片段就會帶進 page table 裏面

Anonymous cache 也是放在 page tbale 裏面的??多想想？？

- 預先讀取 read-ahead

## Recovery

**老師覺得這部分很重要**，2000 年以前在意 performance，之後有很多研究在研究 consistency
老師說研究所開的課才會強調這部分
檔案讀寫到一半斷電了怎麼辦？主記憶體是揮發性的

基本上是在顧及 file system 的結構，不擔心用戶資料丟失（那部分交給 user application 處理 ）

Unwritten 的資料就丟失掉了，產生 structural inconsistency 我 file system 結構就壞了

在 ext4 要新增一個檔案
Allocation bit map 表示為一
做一個 Inode pointer 指向後面的
Data block
還要指向 Directory
這一大串都要成功才算完成，而且檔案也不會立即就寫到磁碟上，這時候突然斷電了，變成爛攤子有些改了有些還沒改**這就是 consistency 的根源**

以 Unix 有 fsck ，Windows 有 scan disk

Super block 一開始會把 dirty bit 設為 1 ，正常 unmount 會把他改為 0，因此今天開機作業系統看到他是 1 就會懷疑是不是出事了，因此透過上面提到的工具去試圖猜測、修復，會需要很久且隨著磁碟越來越大，檢測就要更久

### Journal...

**老師說可見很重要**

### Write Ahead Logging

- Journaling 的其中一種做法 其他還有 (WA2, rollback....)

1. 檔案寫入產生的 dirty data 集合加入到一個 transaction，會在適合的時間關閉
2. 搜集完畢關閉後開始寫到磁碟上 journal Log write operaitons 資料會先寫到 journal
3. 完整地寫道 jornal 後再往 file system 寫，
4. jornal 寫入完成之後才開始朝 file system 寫入，寫完才刪掉 journal (如果重開機看到 journal 有東西就當)

### Journaling File Systems

#### Transactions

- **從 database 偷來的一個概念**
- ACID properties
- All or none

#### Journaling

- Based on write-ahead logging (寫入前先寫到一個 journal 上)
- 保證檔案系統結構正確
- **不保證資料不丟失**

#### 斷電

- 掃一遍 journal
- 有完整的話就重做
- 不完整的就丟掉

好處 : crash recovery very fast
壞處 : 寫會發生兩次，一次是從外部寫到 journal ，一次是 journal 寫到 file system

- 解法 :
  - FS write in background,
  - 只 jorunal meta data

### Log-Structured File System (LFS)

- benefit : optimized random write, easy to recovery
- drawback : 有 compact overhead, 還有可能要用更多的 RAM 來做檔案的 mapping (?)
- need to notice that，不要把他裝到太滿!

原本 JFS 前面一段是 Journal 後面才是 file system ，他是一個環狀的 buffer，放一個一個 transaction
log-structured 就直接全部都是 journal (treat the entire disk space as a single logging area) 就不用再兩段 copy back 了

主要的目標是想要提升 **random write** 的效能，那時候預測 RAM 會變很便宜因此不用處理 random read ，他把 random write 轉換成 out-of-place updates (不在原地更新) 把要寫入的東西不管三七二十一，搜集起來找一塊空間就放進去了，因此寫入就會一直往後**連續的**寫，要找到這個修改過後的檔案可以再透過一個 Inode 紀錄指過來，

Example : NILFS2, F2FS for Android devices
老師說很多先進的技術像是 SMR、flash 都不太能在原地更新，

#### Compaction - garbage collection (老師說 memory 也有提過)

一個 log-structure FS 的 overhead
![[Pasted image 20211222111910.png | 500]]
他是環狀的 buffer 如果修改過後的檔案都放到後面了，前面都是 garbage ，可以把這些破碎的空間往後移動集中

#### Recovery in LFS

![[Pasted image 20220111162811.png | 500]]

因為他是按照時間寫入的，因此可以放一個 check point 在完整寫入檔案的尾巴，在那之後可以直接全部丟掉
