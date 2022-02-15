# Deadlock

[[Operation System 作業系統]]

早期的作業系統是會自動處理 deadlock ，但人們發現由 OS 來處理成本太高，而且這表示程式撰寫本身有問題因此近代的作業系統不處理 deadlock，programmer 自己要處理好，那為什麼我們現在還要學習 ? 因為作業系統的一支 RT(real-time) OS 例如武器系統、自駕車、工廠... 如果發生 dead-lock deadline 絕對趕不上，可能因此就撞車了，因此是非常嚴重的問題

```c
void *do_work_one(void *param){
	pthread_mutex_lock(&first_mutex);
	pthread_mutex_lock(&second_mutex);
}

void *do_work_two(void *param){
	pthread_mutex_lock(&second_mutex);
	pthread_mutex_lock(&first_mutex);
}
```

### Deadlcok Characterization 充分條件必要條件

如果 Deadlock 發生則以下都為真，`deadlock -> c1 ∧ c2 ∧ c3 ∧ c4 ≡ ¬ (c1 ∧ c2 ∧ c3 ∧ c4) -> ¬deadlock`，因此以下都是對付 deadlock 的武器

- c1 : Mutual Exclusion 互斥，我放下了你才能拿
- c2 : Hold and wait : 我會佔到其中一個等另一個資源
- c3 : No preemption : 我不會被迫放棄我拿到的資源，一定是鎖住用完開鎖離開，不能中間突然換人
- c4 : Circular wait : 循環等待

### Resource Allocation Graph

![[Pasted image 20211110103735.png | 200]]

- 上圖表示， P1 拿了一個 R2 想要 R1 但是被 block 住，R1 是 P2 在使用
- P2 拿了 R1 R2 但是拿不到 R3 被 block
- P3 想要 R2 但是也拿不到
  可以在圖上找到一個 cycle (circular wait)

#### With Cycle but no deadlock

![[Pasted image 20211110104249.png | 200]]

#### 會有 deadlock 的條件

- 一份 resource deadlock if and only if cycle `deadlock <-> there is cycle`
- 多份 resource `deadlock->there is cycle`

### Handling Deadlocks

- `prevetion` 讓系統不會進入 deadlock
- `detection` 一但 deadlock 就去偵測及 recovery
- `ignore` : 現在的主流做法，因為太昂貴或是改變軟體的 behavior 提升寫程式的複雜度

### Deadlock Prevention

- Mutual Exclusion : 沒機會，一定是這樣
- Hold and Wait : 原本分開鎖分開拿得多個資源合併成一個，這樣就是全拿全不拿，但是很不建議因為 process 的平行度會降很低，因為可能最後變成 100 resource 當成一組要用裡面其中一個都要整個鎖造成 `low concurrency`
- No Preemption : 如果 P1 拿著 R1 在等，別人要 R1 `P1` 就會被犧牲，把 R1 釋放出去，並且等到 R1 再度可用的時候被叫醒，需要做一個快照來恢復 P1 狀態 (舉例來說 R1 是記憶體，P1 也都放資料進去了結果突然被搶走)
- Ciruclar wait : 如果規定一定只能遞增順序申請資源，先拿 R1 才能去要 R2， 不能反方向操作就不會形成循環的態勢，但是這樣也會造成寫程式很麻煩

### Deadlock Avoidance

作業系統事先得知要用的資源等等額外信息，就可以確定說要不要分配給他

### single instance 處理方法

#### check request

系統會去檢查一個 request (P1 想要 R1)會不會造成死鎖的危險，會的話就擱置這個請求
![[Pasted image 20211110111344.png | 300]]

實線表示正在用，虛線表示發出 request 想要用，P2 想要拿 R2 但是作業系統發現給他就慘了，因此會讓他等到 P1 用完，至於怎麼在圖裡面去 detect 有沒有 cycle 則是離散數學裡面的招式

#### Highest Locker's Protocol in RTOS

老師說挺重要的，碩果僅存真正有 real time OS 在用要能看懂
![[Pasted image 20211110111927.png | 500]]

上半部是會 deadlock 的情況，L 鎖住紫色，H 鎖住紅色想要紫色，回到 L 想要紅色

首先要先分析紅色跟紫色兩種資源，紅色紫色都會有 H L 兩種 Process 會想要用，

下半部解法操作 Low 開始執行所到紫色，因此他的 priority 被提升到比 H 還要高，因此不會發生 preemption 拿到紅色又變更高，用完紅色紫色 priority 就掉下來了並且被 H 拿回去用

再從在邊回顧 `pthread_mutex` 誰去鎖就是誰去解，為什麼是這樣設計呢? 因為你去鎖 / 解我會把你的 priority 提升/恢復，如果是別人來解就世界大亂了

但我的想法是如果今天順序換過來， low 拿到紅色，結果這樣還是沒有超過另一個? 或是原本 high 拿到紅色很高，結果換 Low 拿到紫色反超過他？ [延伸閱讀](https://hackmd.io/@pL5jdumIRiOW0W1t54kNcQ/By4Gk23pe?type=view)

### N instance 處理方法

老師說提出是 197x 年，但沒有真正實作，但是考試還是會考ＸＤ

#### Banker's Algorithm

![[Pasted image 20211110113110.png | 300]]
把作業系統當作錢莊，大家來週轉借錢還錢，錢莊不能倒閉

- 用 Banker 來判斷 unsafe 也有可能不會 deadlock，更寬鬆一點，但他不會讓你進入 unsafe state 寧可錯殺

![[Pasted image 20211110113223.png | 500]]
程式一開頭就要先 claim max 多少等等

- Allocation 表示目前手上有多少 e.g. P2 現在有 3 個 A
- MAX 表示最多會用到幾分 (必然大過 Allocation)
- 把 max - allocation 就可以得到 Need

![[Pasted image 20211110113422.png | 500]]

P0 要太多了，P1 可以給他我可以先借給他然後讓他把手上的兩個 A 還我，我變成 5 3 2 ，這個時候就可以去服務 P3 我手上變成 7 4 3 現在就可以去服務 P0 P4
因此 <P1, P3, P4, P2, P0> 就是一個 safe sequence 找到一個這樣的 safe sequence 就可以說這個`系統是安全`的

但有可能第一個人要的不多我可以給他，但是我借給他也拿到他還給我的資源之後，還是沒有足夠的資源去服務別人

Why all processes make their largest requests (MAX-Allocation = Need) in the checking procedure? 為什麼都要用最大的 largest request (Need) 來處理呢？ 因為借錢還錢如果用最大的量來算的話一定是最困難的，如果連這樣都可以過表示一定是安全的
