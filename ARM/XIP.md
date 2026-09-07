XIP = eXecute In Place。

意思是 CPU 直接對外部 SPI NOR Flash 發出讀取指令(如 Fast Read/2Read/4Read),把 flash 裡的內容當成一般記憶體位址空間讀出來,一個 instruction fetch 一個 instruction fetch 地即時抓,不需要先把整個程式搬進 RAM 再執行。

對應到 SNC7360 的規格,SPIFC(Serial Flash Controller)那節就寫了它支援 Read/FastRead/2Read/4Read 等命令模式,並把 SPI Flash 映射在 0x60000000~0x6FFFFFFF 這個 256MB 位址區間(見 §4.2 Memory-Map),CPU 對這個範圍做 instruction fetch 時,底層硬體會自動轉成 SPI 命令去外部 flash 顆粒讀資料回來——這就是 XIP。

這也是為什麼我上一則訊息提醒:就算完全不透過 cache,CPU 本來就能靠 XIP 直接從 0x60000000+offset 執行程式碼,所以單純「執行 NOP 沒有 crash」這件事不足以證明 cache IP 真的有介入、有做 line fill/hit,因為不管 cache 有沒有作用,XIP 路徑本身就能讓這段位址正常執行。
