# File

Below are all files :

- normal file
- Direcotry files
- Links
- Device Files (device node /dev/)

### File Attributes

- Name
- Identifier : 內部作業系統給他的 id (Linux 給的是 inode number 每個檔案唯一的識別號)
- Location : pointer to file place on device (磁碟位址...)
- Size
- Protection/Permission
- Time

### File Types

![[Pasted image 20211208114229.png | 400]]

### File Access

![[Pasted image 20211208114649.png  | 400]]
想要 Random Access ?

- fseek()
- mmap()

### File operators

fopen()
fclose()
getc()
..

#### 為什麼一定要先開 `fopen()` 再讀寫 ?

因為有很多內部用的 data (meta data) 在開檔案的時候要先把他從磁碟讀入並且 cache 起來，接下來的操作才會有效率

- file pointer
- file open count
- disk location on file
- access rights

#### fopen(): Binary or Text

- fopen("abc.txt", "r+t")
- fopen("xyz.mp3", "rb")

當打開的檔案不是人眼可以看懂的二進位資料就會用 rb ，如果是 text 最好用 text 模式打開，因為有些小細節作業系統實作不一樣 e.g.

- Linux 換行是 `\n(OA)` Windows 是 `\r\n (0D 0A)`
- Translate Ctrl-Z into EOF
- Possibility of filer out the MSB(only 7LSBs uesed)

### Device Node

- 在 `/dev` 下面
- major, minor number 是對應到驅動程式
- 使用 `mknode` 並且給他 device major, minor number (歷史包袱) 就可以創建一個新的 device
  - M8M0 -> sda 第一顆整個硬碟
  - M8M1 -> sda 0 第一顆硬碟的第一個 partition 磁碟分區

當我們 open divce node /dev/sda0 ，作業系統會先去查 driver table，去看到 M8M0 然後拿到對應的驅動程式，因此對 device node 就是一個跳板 read/write 基本上就是在跟驅動程式溝通，也因此 device node permission 只有 root 可以 open (owner 是 root) 不然隨便一個 user 都可以使用

### directory file

- search
- 可以看到操作的話就是先 open 然後 read...

### 樹狀目錄結構

- CWD current working directory **是一個環境變數 (per process)**
  - `.` and `..` (目前的上一層)
- 如果不給詳細路徑就會從 cwd 找?

### File Aliasing (Link)

Link 也是一種 file
![[Pasted image 20211210154000.png | 400]]
Linux example :
`/usr/bin`, `/usr/local/bin` 裡面都有同樣的程式，透過 link 以達到不重複存兩份

#### Soft link (symbolic link)

會分本尊跟分身

- 他就是一個 file，看得出來是一個 Link : ls -l 會看到權限開頭是 L
- 裡面儲存的是 `target path`, 做的是一個字串代換的操作，打開，把裡面的資料拿出來
  跟檔案系統是沒有關係的，我可以用一個 softlink 連接 network file system, CD rom.....

- Usage :
  - UNIX : ln -s [target][link]
  - Windows (NTFS) :junction.exe [link][target]

#### Hard link

比較特殊了，是檔案系統裡面的連結，連結到**磁碟上儲存的位置**

- 跟檔案系統有相依性，不能跨 file system
- 奧妙就是在 UNIX 根本 **看不出來是不是 link 誰是本尊誰是分身**
- unlink() to delete file

- Usage
  - UNIX: ln [target][link]
  - Windows : fsutil hardlink create [link][target]

### Problems with Aliaing

File link 是一種方便的工具但他也帶來一些麻煩

- **備份** : Hard link 上

  - 一顆目錄樹底下很多個 file 因為要備份想要 copy 出來，如果是 soft link 他本身也是一個 file 就把 softlink copy 過去，但如果裡面有兩個 hard link 指向同一個 file ，我們根本不知道他是一個 link 就會把 hard link 底層的資料各自複製一份 duplicate
  - ![[Pasted image 20220110201259.png]]
  - `cp -a` or `rsync` 去努力幫我們處理

- Loop cause by hard link :
  - 如果有 directory link 有可能導致無窮迴圈， UNIX 是不允許創建 directory hard link
    - Modern UNIX 基本上預設不讓你 link 到 directory (下一些參數還是可以)
  - 每次要創就去爭側有沒有回圈不實際 => 很難爭側 butterfly graph
- Loop cause by soft link :
  - Linux : Keep time-to-live counter (40) 走一層減一走完就當作，減完就 跟網路封包傳遞有點像誒
  - Winodws : Limiting the pathname length (老師說可以試試看創一個名字超長的資料夾，往下一直這樣做，幾層下去就走不下去了)

**老師說解決方案要看一下呦**

- **Delete** : Acyclic-Graph Directories
  - 如果兩個檔案 link 到同一個地方，連到的地方被刪掉了會產生 dangling pointer
  - soft link (symbolic link) 解決方案就是不處理，因為 symbolic link 只是一個 file 儲存 target destination，而今天想要打開 target destination 會發現誒！不見了，所以就可以跟我說檔案找不到，不會產生系統層面的問題
  - hardlink 比較麻煩 : link count 的方式，因此要 link count 變成零才真的可以刪掉，揭曉因此我們沒有 delete 而是 unlink()，如果今天只有連到一個地方 unlink() 之後歸零就會殺掉

### Soft link vs hard link

- can can't span over file system

### File - system Organization

fdisk -> format (mkfs) -> mount()

#### fdisk 分割

![[Pasted image 20220110202516.png]]

- 把磁碟做分割，以便讓磁碟不同區域做不同的用途使用，個一塊空間割出幾個 partition (可以自由設定成 ext, swap, FAT...)

- 磁碟最開頭有一個 MBR (master boot record)開機啟動第一章有講過，內含一個 boot code + 非常小的 partition table (64byte) 紀錄底下的 partition 大小分別又是什麼的檔案系統，分割的結果就是在這個 table 上做好記錄

#### Disk Formatting 格式化

- 高階格式化/邏輯格式化/make a file system : 把檔案系統的表格準備好，file system metadata 初始化寫進去 (根目錄的，配置圖...)

  - write file system metadata
  - media scan (optional) 有加上這個才會很久 (把資料寫進去再讀出來看有沒有問題)

- 低階格式化 :
  - Remapping bad trac ks to spare tracks (把磁碟裡面一些不穩定不可靠的區塊隱藏起來不要給作業系統知道)
  - 沒事不用去做，一般來說 user 不需要做，有一些以訛傳訛中毒了要去低階格式化

#### mount 掛載起來

將檔案系統與目錄樹結合，掛載點一定是目錄，該目錄為進入該檔案系統的入口

- e.g. mount -t ext4 /users /dev/hda1
- 標注好檔案系統的類型
- 在 partition 中找到檔案系統的 superblock
- 在檔案系統的 naming sapce 中標註 mounting point

- 一個檔案系統一定要 mount 才能被存取
- 一個還沒 mounted 的檔案系統會在 mount point 被 mount

`fdisk` /dev/hda.....分割出 1 2 3
`mkfs` 格式化 /dev/hda/hda1
mount -t ext4 /users /dev/hda1

### Protection

```
drwxr-xr-x@   8 kevinzhang  staff      256 Dec  2 15:44 baidu
drwxr-xr-x@   6 kevinzhang  staff      192 Jun  7  2021 blog
-rw-r--r--@   1 kevinzhang  staff    40960 Aug 11 15:28 bomb.tar
drwxr-xr-x   29 kevinzhang  staff      928 Dec  8 00:35 books
drwxr-xr-x    7 kevinzhang  staff      224 Sep  1 14:18 cs106x

permission / link count /
```

directory link count :

- directory 都會自己指向自己，如果有一個空的子目錄基本消費額就會是 2 (父->a+ a->a)
- 3 (父->a + subdir1->a +subdur2->a) 子目錄會指向自己所在的 directory (也因此 link - 2 = 目錄裡面的子目錄數量) (檔案是不會往回指的)
  第一個字 d : directory / owner / group /

-rwx -rwx -rwx 有三組權限，分別是 owner, group,

#### UNIX File permission Management Utilies

- chmod : 偷懶就 `chmod 777`全開
