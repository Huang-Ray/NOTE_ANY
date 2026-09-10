 scatter file guide · MD
# SNC7360 Scatter File（.sct）知識整理
 
> 目的：整理 Keil/armlink scatter-loading description file 的基礎架構、VMA/LMA
> 定義方式，以及 SNC7360 這個 SDK 架構下，scatter 檔與「實際燒錄進 SPI-NOR
> flash 的 bin 檔佈局」之間如何分工。所有範例位址取自
> `development/verify_fpga_a/icache/snc7360_cache_execute/keil/linker.sct`
> 與其編譯出的 `.map`，實際數值會隨 `ez_config.h` 設定與程式碼大小變動，
> 但架構規則不變。
>
> 延伸閱讀：`claude/SNC7360_verify_pattern_porting_SOP.md`（SNC7360 vs
> SNC7330 執行位址模型差異）、`claude/SNC7360A_spec_summary.md` §7.13
> （Cache 控制器與執行位址模型）。
 
---
 
## 1. 基本架構：三層巢狀
 
```
Load Region  (LR)
  └─ Execution Region  (ER)          ← 一個 LR 底下可以有多個 ER
       └─ Input Section Selector     ← 決定哪些 .o / section 塞進這個 ER
```
 
語法：`名稱  位址  [屬性]  size { ... }`，Load Region 和 Execution Region
共用這個模式，差別只在巢狀層級。
 
具體範例（節錄自 `linker.sct`）：
 
```
ROM_INTERNAL_PRAM   0x00000000        0x10000          ← Load Region
{
    CODE_PRAM        0x00000000       0x10000          ← Execution Region
    {
        *.o (RESET, +First)                              ← Input Section Selector
        *(InRoot$Sections)
        *armlib*        (+RO)
        startup.o       (+RO)
        system_init.o   (+RO)
        snc_drv_system.o(+RO)
        .ANY1           (+RO)
    }
 
    DATA_SRAM   (0x18000000+SRAM_CODE_SIZE)  (0x20000-SRAM_CODE_SIZE)   ← 另一個 ER
    {
        .ANY  (+RW +ZI)
    }
}
```
 
---
 
## 2. VMA 定義在哪
 
**VMA（執行位址）= Execution Region 那一行寫的位址，永遠是。**
 
- `CODE_PRAM 0x00000000 0x10000` → VMA = `0x00000000`
- `DATA_SRAM (0x18000000+SRAM_CODE_SIZE) ...` → VMA = `0x18000000`
- `USER_EXT EXT_LOAD_ADDR EXT_CODE_SIZE` → VMA = `EXT_LOAD_ADDR`
沒有例外：Execution Region 宣告的位址，就是 CPU 執行/存取這段內容時實際
去的位址。
 
---
 
## 3. LMA 定義在哪
 
Load Region 自己的位址（最外層那個）＝這個 Load Region 整包內容在最終
映像檔裡的起始 LMA。例如 `ROM_INTERNAL_PRAM 0x00000000` → 這包內容從
image 的 offset `0x0` 開始存放。
 
但 **Execution Region 本身沒有另外一個「LMA 欄位」**——只有一個位址
（即 VMA）。LMA 怎麼決定，要看有沒有 `OVERLAY` 屬性：
 
### 3.1 沒有 `OVERLAY`（預設，本專案大多數 region 都是這樣）
 
同一個 Load Region 裡的 Execution Region，LMA 是 linker **自動接續排列**
算出來的：
 
- 第一個 Execution Region 的 LMA = Load Region 自己的位址
- 後面每一個的 LMA = 前一個 Execution Region 的 LMA + 前一個**實際編譯出來
  的內容大小**（不是宣告的 size 上限，是真實用掉的 bytes）
實測範例（`snc7360_cache_execute` 的 `.map`）：
 
```
Execution Region CODE_PRAM  (Exec base: 0x00000000, Load base: 0x00000000, Size: 0x000040e8)
Execution Region DATA_SRAM  (Exec base: 0x18000000, Load base: 0x000040e8, Size: 0x000012a0)
```
 
`DATA_SRAM` 宣告的位址 `0x18000000` 只決定 VMA；它的 LMA（`0x40e8`）完全
沒人手寫，是 linker 看到它跟 `CODE_PRAM` 同一個 Load Region、又沒標
`OVERLAY`，就自動接在 `CODE_PRAM` 實際內容（`0x40e8` bytes）後面存放。
 
這正是嵌入式最經典的「RW 資料 initial value 存 ROM/Flash、執行時搬到
RAM」寫法：VMA/LMA 故意不同，但完全不用手寫兩個位址，linker 自己排。
開機時 `__main` 的 scatter-loading 機制（自動產生的 `Region$$Table`）會
把 `.data` 內容從 LMA 複製到 VMA。
 
> **ZI 資料（純 BSS，沒有初始值）完全不占 image 空間、不需要 LMA**，
> 開機時直接在 VMA 位置清零即可。只有帶初始值的 `.data` 才需要 LMA。
 
### 3.2 有 `OVERLAY`
 
```
CODE_SRAM   SRAM_LOAD_ADDR   SRAM_CODE_SIZE          ← Load Region
{
    CODE_CORE1   CORE1_START_RAM_ADDR   OVERLAY NOCOMPRESS   CORE1_CODE_SIZE
    {
        *(core1_image_code)
    }
    CODE_SRAM    CORE0_SRAM_CODE_ADDR   OVERLAY NOCOMPRESS   CORE0_SRAM_CODE_SIZE
    {
        .ANY (+RO)
    }
}
```
 
`.map` 顯示兩者都是 `Load base: 0x60011000`（跟 Load Region 自己的位址
完全相同，不是接續排列）。
 
`OVERLAY` 的意思：**不要用「接續排列」規則，強制讓這個 Execution Region
的 LMA = Load Region 自己的位址**，等於讓多個 OVERLAY region 共用同一塊
load 空間。用途是讓互斥的替代佈局（例如 Core1 / Core0 兩種韌體配置）
共享同一個實體 flash 位置，因為它們不會同時燒同一份 image 使用。
 
> 這也是 SNC7330 舊平台把 flash 內容 remap 到 `0x10000000` CPU 執行視窗
> 常用的技巧（`OVERLAY` 到一個跟 Load 位址完全不同的 Exec 位址）。這個
> `0x10000000` alias 視窗在 SNC7360 上**不存在**，SDK 模板裡沿用這個
> `OVERLAY` 寫法是錯的，`snc7360_cache_execute` 專案裡已把 `CODE_EXT`
> 的這個 `OVERLAY` 移除，改成 Exec = Load 都直接等於 `EXT_LOAD_ADDR`
> （見 §6）。
 
---
 
## 4. Scatter 檔同時在做「記憶體配額檢查」
 
每個 Execution Region 宣告的 size 是一個**上限**，超過就是連結錯誤：
 
```
.\obj\...axf: Error: L6220E: Load region CODE_EXT size (16384 bytes) exceeds limit (0 bytes).
```
 
原因：`ez_config.h` 裡 `EXT_CODE_SIZE` 被改成 `0x0000`，但實際要塞進
`CODE_EXT`/`USER_EXT` 的 `nop_16k.o` 需要 16384 bytes（0x4000），超過宣告
的上限（0 bytes）就報錯。修法：把 `EXT_CODE_SIZE` 改回 `0x4000`。
 
---
 
## 5. Scatter 檔管不到的部分：實際 flash image 的佈局
 
**scatter 檔只定義 VMA/LMA，這兩者都是相對於 armlink/`.axf` 這個編譯產物
的概念，跟「最終要燒進 SPI-NOR flash 的 bin 檔案，實體 byte 怎麼排」是
兩件完全獨立的事。**
 
以本專案為例：
 
```
Load Region ROM_INTERNAL_PRAM (Base: 0x00000000, ...)
  Execution Region CODE_PRAM (Exec base: 0x00000000, Load base: 0x00000000, ...)
```
 
`CODE_PRAM` 的 LMA 就是字面上的 `0x00000000`（跟 VMA 相同，這個 region
沒拆分）。但實際上這段 code 最終要被燒到 flash 的 `0x60001000`（也就是
`ez_config.h` 的 `PRAM_LOAD_ADDR`）。**`0x60001000` 這個數字，在 `.axf`
的 scatter/LMA 概念裡完全不存在**——它只是 `sdk/bsp/sn_m3b/sonix_load_table.c`
這個原始碼裡，`load_table` 這個 struct 初始化時寫死的一個**資料值**
（`.fw_info.PRAM_ADDRESS = PRAM_LOAD_ADDR`），這個 struct 被編進另一個
獨立的 Load Region `SN_LOADTABLE`（LMA=VMA=`0x60000000`）當成一般唯讀
資料存放，armlink 自己完全不知道、也不會用這個數字做任何位址配置。
 
真正把 `CODE_PRAM` 的內容從 `.axf` 搬到 flash image 正確位置的，是
**Keil 專案設定裡的 Post-Build User Command（`AfterMake`）**：
 
```
<AfterMake>
  <RunUserProg1>1</RunUserProg1>
  <UserProg1Name>..\..\..\..\..\toolchain\windows\SonixSNC73xxxMergeTool\SNC73xxx_Merge_Tool.exe
    -info "!L" -crc_gen "!L" -gen_bin "!L" "$Loutput\@L.bin"</UserProg1Name>
</AfterMake>
```
 
`SNC73xxx_Merge_Tool.exe`（來源：`toolchain/windows/SonixSNC73xxxMergeTool/`）
的工作流程（由其內部符號名稱 `SN_M3X_PRAM_EXEC_ADR`、`PRAM_ADDRESS`、
`cal_sn_loadtable_crc`、`save_sn_loadtable` 等可確認）：
 
1. 讀入編譯出來的 `.axf`。
2. 從 `.axf` 的 `SN_LOADTABLE` region 取出 `load_table` 結構，拿到
   `PRAM_ADDRESS`（= `PRAM_LOAD_ADDR` = `0x60001000`）等欄位。
3. 把 `.axf` 裡 `CODE_PRAM`（LMA=`0x0`）的實際內容，複製貼到新產生的
   合併檔案裡，位移到「load table 自己的位址 `0x60000000` +
   (`PRAM_ADDRESS` − `0x60000000`) = `0x60001000`」的位置。
4. 因為搬動會影響 load table 的完整性，重算並回寫 CRC。
5. 輸出最終合併檔 `..._update.__AT_0x60000000.bin`——檔名裡的
   `__AT_0x60000000` 表示這是一個 flat/raw binary，**檔案 offset `0x0`
   對應到燒錄後的實際 flash 位址 `0x60000000`**，之後每個 offset 都是
   「offset + `0x60000000`」＝實際 flash 位址。這份檔案才是真正要燒進
   SPI-NOR flash、給晶片冷開機用的產物。
對照表（本專案目前的 `ez_config.h` 設定下）：
 
| bin 檔 offset | 實際 flash 位址 | 內容 |
|---|---|---|
| `0x0000` | `0x60000000` | `SN_LOADTABLE`（`sonix_load_table.c` 的 `load_table` 結構） |
| `0x1000` | `0x60001000` | `PRAM_LOAD_ADDR`，`CODE_PRAM` 內容（由 Merge Tool 搬過來） |
| `0x11000` | `0x60011000` | `SRAM_LOAD_ADDR`（若 `SRAM_CODE_SIZE` > 0） |
| 依設定 | `EXT_LOAD_ADDR` | `CODE_EXT`/`USER_EXT`（例如本專案的 `nop_16k.o`） |
 
### 5.1 這跟 JLink Debug Download 是兩條不同的路
 
`.uvprojx` 的 `<FlashDriverDll>` 定義了兩個獨立的 JLink flash algorithm：
 
- `SN_M3B_PRAM`：位址範圍 `0x0` ~ `0x10000`
- `SN_M3B_FLASH`：位址範圍 `0x60000000` ~ `0x61000000`
JLink Debug Download 是**直接照 `.axf` 裡各 region 宣告的位址燒錄**——
`CODE_PRAM` 就照字面燒到 `0x0`（用 `SN_M3B_PRAM.FLM`）。這是給偵錯用的
路徑，跟上面 Merge Tool 產生的量產 flash image 完全獨立、互不影響。
 
### 5.2 為什麼 `CODE_EXT`（本專案的 NOP 測試 code）不需要 Merge Tool 轉換
 
因為 `CODE_EXT`/`USER_EXT` 的 LMA 跟 VMA **直接就宣告成同一個 flash 位址**
（`EXT_LOAD_ADDR`），編譯完就已經躺在正確的 flash 位址，Merge Tool 只是
單純把它搬到 bin 檔對應 offset，不需要像 `PRAM`/`SRAM` 那樣做
「內部 RAM 執行位址 → flash 儲存位址」的轉換。這也是
`snc7360_cache_execute` 專案能用「編譯階段直接放 code 進 flash」取代
「runtime SPI erase/program」的原因。
 
---
 
## 6. `snc7360_cache_execute` 專案的 `CODE_EXT` 修正紀錄
 
SDK 模板原本的 `CODE_EXT` 用 `#if(CONFIG_USE_ICACHE)` 分支，搭配
`OVERLAY` 把執行位址 remap 到 SNC7330 舊平台的 `0x10000000` alias 視窗
（SNC7360 上不存在這個視窗）。已修正為固定寫法：
 
```
CODE_EXT           EXT_LOAD_ADDR           EXT_CODE_SIZE
{
    USER_EXT        EXT_LOAD_ADDR           EXT_CODE_SIZE
    {
        nop_16k.o       (+RO)
    }
}
```
 
Exec = Load = `EXT_LOAD_ADDR`（無 `OVERLAY`），直接對應 §5.2 的效果。
 
---
 
## 7. 快速對照表：VMA / LMA 決定規則
 
| 情況 | VMA | LMA |
|---|---|---|
| Execution Region 未標 `OVERLAY`，是同一 Load Region 裡第一個 ER | 宣告位址 | = Load Region 自己的位址 |
| Execution Region 未標 `OVERLAY`，非第一個 ER | 宣告位址 | = 前一個 ER 的 LMA + 前一個 ER 實際內容大小（接續排列） |
| Execution Region 標了 `OVERLAY` | 宣告位址 | = Load Region 自己的位址（不接續排列，強制歸零/共用） |
| ZI（純 BSS，無初始值） | 宣告位址 | 不需要，不占 image 空間 |
| Scatter/armlink 完全管不到的：最終燒錄 bin 檔的實體佈局 | — | 由 `sonix_load_table.c` 的資料 + `SNC73xxx_Merge_Tool.exe`（Keil `AfterMake` user command）決定，與 armlink 的 VMA/LMA 概念無關 |
