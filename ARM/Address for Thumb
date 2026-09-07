# 6. 技術背景補充:`test_addr` 為什麼要 `| 1`(Thumb 位址標記)
 
`pattern_test()` 裡:
 
```c
uint32_t test_addr = (0x10000000 + test_offset[i]) | 1;   /* +1: thumb mode */
void (*pFunc)(void) = (void (*)(void))test_addr;
pFunc();
```
 
這個 `|1` 跟 I-Cache 的位址映射邏輯無關,是 Cortex-M 架構本身呼叫慣例的規定,值得記錄下來
(SOP 2.5 節也有收錄,因為之後任何測項只要用函式指標動態呼叫一段自己算出來的位址,都會
碰到同一件事):
 
- Cortex-M3 只支援 Thumb/Thumb-2 指令集,沒有傳統 32-bit ARM 狀態。
- `BX`/`BLX Rm` 用目標位址的 **bit0** 決定切換到哪個執行狀態:1=Thumb,0=ARM。
  Cortex-M3 沒有 ARM 解碼器,bit0=0 時 CPU 連指令都還沒抓就會先噴 UsageFault
  (`FAULT_USAGE_INVSTATE`,`cortex_fault_e` 編號 14)。
- **平常寫 `void (*p)(void) = my_func;` 不用管這件事**,因為 `my_func` 是連結器認得的
  Thumb 函式符號,連結器產生符號位址時就已經自動把 bit0 設成 1(向量表裡
  `Reset_Handler`/各個 IRQ Handler 反組譯出來也都是奇數位址,同一個道理)。
- 但 `test_addr` 是我們自己用整數運算湊出來的(`0x10000000 + test_offset[i]`),不是
  來自任何函式符號,連結器不會幫忙標記,所以要自己 `|1` 補上。**判斷準則是「這個位址
  是不是直接來自函式符號本身」,不是「看起來是不是函式」**——就算寫死的常數剛好等於
  某個函式的真實位址,只要是用常數/整數運算方式賦值,一樣要自己補這個 bit。
- 這個 bit 只影響「用哪種狀態解碼」,不影響「實際從哪個位址抓指令」——CPU 執行
  `BX`/`BLX` 時會把 bit0 清掉才當成真正的抓取位址使用,所以 `pFunc()` 實際執行的位址
  還是乾淨的 `0x10000000 + test_offset[i]`,不會因為多 OR 1 而多讀到旁邊那個 byte。
