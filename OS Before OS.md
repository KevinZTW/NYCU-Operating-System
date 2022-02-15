### Computer Startup 開機流程

program counter 指向要執行指令的位址，重開機的話

- pc 指向 FFFF:0000 最高位址，主機板上的 ROM / BIOS 內容被映射到那個地方
- 磁碟開機的話 BIOS 會負責從磁碟上複製一段 512 Byte 的 MBR 載入到記憶體裡面
- 執行 MBR master boot record 任務是載入 Boot manager
- 把 Boot manager 也載入，
- Boot manager 把選擇的 OS 做載入
