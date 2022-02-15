# OS Synchronization

[[交大張立平作業系統 作業三]]

上課聽老師講好像都很簡單，實際做就傻掉，寫個 Pseudo code 都寫不出來，實踐出真理的單元

## Background

複習一下 之前有講到用一個 ring 環狀的 Share memory ，producer 就環狀往裡面放， consumer 往裡面拿，這樣放了一圈怎麼知道是空的還是滿的 ?
有講到可以犧牲一欄位，留一個不要放，這樣空的時候 in 跟 out 就在同一欄位

### 可不可以用 item counter ?

(我自己也想這樣做，用 n = 123 表示裡面有幾個 producer +1, consumer -1)
producer

```c
while(true){
		while(count == BUFFER SIZE){
		//do nothing
		}
}
//consumer
while(1){
	while(count ==0)
		;//do nothing
	count --
}
```

#### Race Condition

count ++ could be implemented as below， 注意 context switch 可能在執行任何一步的時候發生

```
register1 = count
register1 = register1 + 1
count = register1
```

count-- could be implemented as

```
register2 = count
register2 = register2 - 1
count = register2
```

Consider this execution interleaving with “count = 5” initially:

```
S0: producer execute register1 = count {register1 = 5}
S1: producer execute register1 = register1 + 1 {register1 = 6}
S2: consumer execute register2 = count {register2 = 5}
S3: consumer execute register2 = register2 - 1 {register2 = 4}
S4: producer execute count = register1 {count = 6 }
S5: consumer execute count = register2 {count = 4}
```

##### Race example

![[Pasted image 20211020113000.png]]
可能在判斷完 safe == true 進去之後瞬間被 thread 2 打斷
問題顯然易見是 safe 這個值應該是不可打斷的，race 的 bug 極難處理可能跑一百次出現一次錯誤，唯有一開始就觀念正確
![[Pasted image 20211020113344.png]]

#### busy waiting

剛剛的方法除了 race condition 還有 busy waiting 的問題，(consumer 一直等阿有沒有任務？阿沒有...)

## Ctitical Section

### 解法必須滿足三個條件

#### Mutal Exclusion

如果有人來，那就不能有其他人來動
If process Pi is executing in its critical section, then no other processes can be executing in their critical sections.

#### Progress

如果沒有人要進去，而我要進去，那我就一定可以進去
聽起來是廢話，但其實有一種錯誤的解法 - 輪流，可以達成 mutual exclusion 以及 bounded waiting 但是 p1 在忙別的死不運行，那 p2 想要進去就得一直等

#### Bounded Waiting :

一個 bound 像是房間一樣，一但我不能進去，我必須給他一個預期的等待，等 n 次就會讓他進去

There exists a bound, or limit, on `the number of times` that other processes are allowed to enter their critical sections after a process has made a request to enter its critical section and before that request is granted.

## Synchronization Harware

### 關閉中斷 Interrupt Disabling

在單核心 + kernel 才有效

因為 Timer, I/O completion 是打斷的原因，因此 Mask interrupts prevent the running process from being preempted，在 x86 有 CLI / STL

- 問題 1 : priviledge
- 問題 2 : 多核心無效，我有兩個 CPU 在 race

這個解決方案可以用，但是這些中斷遮罩是特權指令，在 user space 不能用！kernel 裡面用才可以，所以會看到 embedded 會看到他們這樣自己實作，在 x86 : 什麼事也不會發生，others : 產生 Trap 到 OS 去

### Test and set or swap

Mutal exclusion 滿足
Progress 滿足
bounding waiting 沒辦法保證，可能 starvation
雖然沒有三個都滿足，但是作業系統確在用

就是一個 memory 位址，把 true 寫進去、裡面東西拿出來 (這一段操作是 Atomic 的，不然會出問題)

```c
boolean TestAndSet(boolean *target){
	boolean rv = *target;
	*target = TRUE;
	return rv;
}
```

如果原本是 true，表示原本鎖住了，我再鎖定並且發現啊鎖住了

如果原本是 false，我去 test and set 我把 true 放進去並且拿到 false 表示鎖起來並且發現誒！原本沒鎖誒

```c
	while(TestAndSet(&lock));//do nothing

	//critical section

	//finish critical part
	lock = FALSE ;
```

剛剛的操作其實就是交換，所以其實可以用任意數值意思一樣

```c
	key = TRUE;
	while( key = TRUE)
		Swap(&lock, &key);

	//critical section

```

![[Pasted image 20211022155017.png]]

#### 好處

單核心 多核心都可以用
user, kernel mode 都可以

#### spin lock 壞處

#OS ⚡️

##### 效能問題

要一直 spin 把 cpu 佔著來繞，單核心的狀態下別人也沒辦法解鎖，只能 spin 到時間用完被 timer interrupt

##### starvation 問題

他的 contention 是無狀態的, process starvation 是可能的

有 Pi, Pj 兩個 process，如果 Pi 拿到了用完之後又立刻再把他鎖起來，Pj 幾乎是沒有機會拿到的

#### Bounded-waiting solution based on TAS/SWAP

基本沒人在用

### Atomic Instructions

指的是說一系列的 instruciton 過程不會被中斷，會通通一次執行，沒有做到一半有其他人來 interrupt 變成其他人做的狀況

再多處理器環境下，CPU 執行 atomic instruction 的時候對 target memory 有 `exclusive access`，什麼意思? 指的是當 CPU 從 memory 拿了一個變數時其他 CPU 都不能動他

`Cache coherence protocol` 兩個 CPU 各自有一個 cache ，從 memory load 3 進去一個加一，一個減一，各自的快取變成 2, 4 ，要解決這個問題的方式

## Synchornization Software

軟體的方式老師說只有在兩個 process 才會 work 而且超複雜，他不打算講

## [[semaphore 信號機]]

抽象的 API ，幾乎所有的作業軟體都有這個機制，他本身底層也是靠剛剛的 `interupt disable` `test and set` 來實作，裡面有一些計算機架構有關的神秘東東但是我們暫時不用擔心不用管

### 小例題

#### e.g. 1 有酒食先生饌

```c
kevin(){
	wait(m);
	eat();
}

professor(){
	eat();
	signal(m);
}
```

#### e.g. 2 我跟女朋友約去看電影，先到要等

```c
kevin(){
	signal(kevin);
	wait(kelly);
}

kelly(){
	signal(kelly);
	wait(kevin);
}
```

#### e.g. 1000 個人要去圖書館 一次 50 人

```
sem_t room = 50;
stduent(){
	wait(room);
	study();
	leave();
	signal(room);
}

```

#### e.g.A DMA controller supports four channels of data xfer

- 一開始會想做四個信號機 s1 s2 s3 s4 一個一個 wait? 但是這樣做 wait s1 就卡住了，正確的邏輯不該卡住
- 應該要有一個信號機保護表格，另一個初始為 4
  ![[Pasted image 20211027111856.png]]
- 兩個 wait 調換可能會有 dead lock 的問題

#### 男女不同房，最多一次三人

```c
sem_t mutexG = 1;
sem_t total = 3;
sem_t sexlock = 1;

void* Girl(void* pool) {
  dispatch_semaphore_wait(gmutex, DISPATCH_TIME_FOREVER);
  if (girlNumber == 0) dispatch_semaphore_wait(sexlock, DISPATCH_TIME_FOREVER);

  if (girlNumber == 3) {
    dispatch_semaphore_signal(gmutex);
    return NULL;
  }

  girlNumber++;
  // dispatch_semaphore_wait(total, DISPATCH_TIME_FOREVER);
  printf("New girl comes in! Now there are %d\n", girlNumber);
  fflush(stdout);

  dispatch_semaphore_signal(gmutex);

  // do something
  printf("girl eat dinner\n");
  fflush(stdout);

  // leave
  dispatch_semaphore_wait(gmutex, DISPATCH_TIME_FOREVER);
  girlNumber--;
  if (girlNumber == 0) dispatch_semaphore_signal(sexlock);
  printf("girl leave now there are %d\n", girlNumber);
  dispatch_semaphore_signal(gmutex);
}


```

#### 托嬰中心，一個大人三個小孩

```c
S = 0
M = 1
adult(){
	signal(S);
	signal(S);
	signal(S);

	wait(M);
	wait(S);
	wait(S);
	wait(S);
	signal(M);
}

child(){
	wait(S);

	signal(S);
}


```

#### 練習的方法

自己問一下，各個題型中我 wait 調換會怎麼樣 ?

### 經典問題 - 以信號機搞定

#### Bounded-Buffer Problem

- 有 producer consumer 有人放有人拿，開頭講過的 buffer 有 n 個....

1. 假設有 `N-x` producers `x` consumers
2. 需要一個 mutex 保護任務隊列 init 1
3. full init : 0
4. empty init : N (表示有 N 個 producer 可以一起衝進來放東西)

```c
sem_t buffer = 0;

producer(){
	//produce an item
	wait(empty);
	wait(mutex);

	//add item into the buffer

	signal(mutex);
	signal(full);
}

consumer(){
	wait(full);

	wait(mutex);
	//take out the item
	signal(mutex);
	signal(empty);
}
```

- 如果改成 `mutex` 在外 `empty`在裡面會發生什麼事? => `dead lock!`

#### Readers-Writers Problem

基本大部分作業系統都有 read - write lock 可以用，基本就是在信號機上再包裝一層

會有一個資料在多個 concurrent 的 processes 之間 share，裡面有多個 reader (only read wont update) 多個 writers (can read and write)
希望 reader 不排他，有其他 reader 可以進來，但是 writer 排他一次只能有一個人在寫，且有 writer reader 就不能來反之亦然

![[Pasted image 20211027113630.png]]

- 把 reader 設成一個 group
- semphamore `mutex` 設置為 1 來保護 reader 之間共用的資料
- semphamore `wrt` 設置為 1
- int readcount 設置為 0 (因為第一個來的 reader 要代表大家去搶 mutex lock)

```c
writer(){
	while(1){
		wait(wrt);
		//writint is performed
		signal(wrt);
	}
}
```

```c
reader(){
	while(1){
		wait(mutex);
		readcount ++;
		if (readercount == 1) wait(wrt);
		signal(mutex);

		//reader read the data

		wait(mutex);
		readcount --;
		if (readercount == 0) signal(wrt);
		signal(mutex);
	}
}
```

以上解法有一個問題， reader 可以一直進來而且在這個情況下， `wrt` 的鎖不會交出來 `writer` 會 starve

#### Dinining Philosophers Problem

![[Pasted image 20211027114804.png]]

有五隻筷子，我要拿起左邊右邊的筷子才能吃飯

semaphore chopstick[5] initialized to 1

解法
有一個人先拿左邊的筷子
只允許 n-1 個同時拿筷子
一次拿一雙筷子

#### Sleeping Barber Problem

理髮店裡面有幾張椅子，滿了客人就不來了，理髮師沒有客人就會睡覺，有的話就會幫他們剪頭髮

- 是事件 所以信號譏為 0

```c
barber(){
	while(1){
		wait(customer);
		//cut customer's hair

		wait(chair);
		chairNumber ++;
		signal(barber);//表示說我可以幫你剪頭髮了
		signal(chair);
	}
}

customer(){

	//check if have space
	wait(chair);

	//get and occupied space
	if (chairNumber < 0){
		signal(chair);
		return;
	}

	chairNumber--
	//leave if no space

	signal(customer);
	signal(chair);//放開椅子的鎖

	wait(barber);//理髮師有沒有空


}
```

- 測試自己的理解，如果 `signal(customer)` `signal(chair)` 順序調換會發生什麼事?
  - 可能會導致客人超車? 不過原本的看起來客人也會超車?

## Mutex

### API

`pthread_mutex_xxx()`

- pthead.h (mutex 只有 thread 可以用)

`sem_xxx()`

- sephamore.h
- process, thread 都可以用

### Mutex vs Sepahmore

跟初始值為 `1` 的信號機很像，功能也差不多，但是信號機誰去 `wait` 他可以是不同的 `process` 去 `signal` 他，mutex 則是 `同一人` 才能鎖起來跟開鎖

## Monitor

### JAVA monoitor

monoitor 裡面保護的整個 block 一次就只會有一個程式進去

- Use the keyword synchronized
- Condition variables
- or wait(),
  ![[Pasted image 20211029160003.png]]
