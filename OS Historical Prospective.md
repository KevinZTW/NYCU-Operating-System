# Syllabus&Ch0: Historical Prospective

書只會教到概念，不會教你實作一個作業系統出來，課程的作業想要彌補 concept 沒有實作的部分

Nachos MP (machine problem)

educated operation system develop by UC Berkley

課程 system call, memory managment, process scheduler, file system

## System Category

## Mainframe systems

mainframe 體積很大的意思用來形容第一代電腦, its one of the eraliest computers. 現在單一處理大量運算的機器我們仍然稱作 mainframe, better reliablility & security

- slow io device, card reader/printer, tape drivers
- Evolution
  - Batch
  - Muti-programming
  - Timr-sharing

#### Batch systems

Single user, single job

一次只能處理一個程式，OS simply transfer job from one job to next
Memory space : operation system / user program area (only these two pieces)

Draw backs

- One job at a time
- No interaction between user and jobs
- CPU is often idle (閒置) I/O speed <<< cpu speed (at least 1:1000)

#### Muti-Programming systems

mutiple program

> 解決的問題 : 讓 CPU 的使用率提高

- Overlaps the I/O and computation of jobs
- Spooling (Simultaneous Peripheral Operation On-Line)
  - I/O could be done without CPU intervention
  - CPU just need to be notified when I/O is done

Job scheduling : 第一個決定是要從 Job pool load 哪些資料到 memory?

- Job pool : Disk
- Memory space : os / job1/job2/job3

CPU Scheduling : 在 memory 上的 job 哪些先放到 CPU 做運算？

##### OS Task

- Memory management
- CPU schedulign
- I/O system

#### Time-sharing System (Muti-Tasking System)

mutiple people mutiple program

> 想解決的問題 : 讓很多人同時使用電腦，而且每個人都覺得自己是唯一的使用者，且跟系統之間有互動

在 Muti-Programming 的 System 上雖然可以一次 load 很多資料到電腦上同時做多個 I/O 操作 跟 cpu 運算，但是在運算是還是 by batch 的

- An interactive system provide direct communication between the user and the system
- Mutiple users can share the computer simutaneously
- switch job when finish, I/O, a short period of time

##### OS Task

- Virtual memory : 讓很多人同時輸入資料
- File system and disk management :
- Process synchronization and dead lock : support concurrent execution of programs

## Computer system architecture

### Parallel System (Muti Processor)

差多個 cpu / gpu
Aka a `mutiple processor` or tightly coupled system - more than one CPU - Usually communication with shared memory

Purpose

- Throughput, Economical, Reliablility

#### Symmetric Multiprocessor System (SMP)

- 現在家裡的個人電腦大多都是這種
- Each processor runs the same OS
- Require extensive sychronization to project data integrity

#### Asymmetric Mutiprocessor System (不對等)

- Each processor is assigned a specific task
- One Master and mutiple slave CPU
- 超級電腦

### Muti-Core Processor

![[Pasted image 20210410203542.png]]

- A processor has mutiple CPU on same chip
- On chip communication is faster than between-chip communication
- They would make share that the data on different processor's cache are the same

### Many-Core Processor

一個 GPU 可以塞進去幾百個 CPU，但是他們一次只能做一個 instruction 適合做 metirc 的 array 計算 無法做一堆不同的 if else...

- Nvidia general- purpose GPU
- Intel Xeon Phi 介於傳統 GPU 跟 CPU 之間
- TILE64

### Memory Access Architecture

- Uniform Memory Access (UMA)

  - 任一 CPU Memory access time 必須要相同
  - 大多數電腦的類型

- Non-Uniform Memory Access (UMA)
  - 當 CPU 大量增加時 Memory 變成瓶頸
  - CPU 切成多個區域，一個區域分一個 local memory 其他 memory 還是可以連但是會比較慢

### Distributed System

- processors communicate with others through various communication lines (I/O bus or network)
- easy to scale to large number of nodes

#### Purpose

- Resource sharing
- load sharing
- Reliability

#### Architecture

- Peer to peer (ppStream, bttorrent, internet)
  - decentralized
- Client to server (FTP )
  - easier to manage and control resources
  - server become bottleneck and signle failure point

### Distributed System - Cluster

- Shared closely through local area network

## Specail Purpose System

### Real-Time System

Well defined fixed-time constraints

- Real time doesnt mean speed, but keeping deadlines
- Guranteed response and reaction times
- Weapon system, medical imaging system, scientific experiment

###
