---
tags: early_boot, Concept
---
# x86 Paging 

## Memory management in long mode

在**protected mode** 下，memory access 會受到 segmentation 機制影響。CPU 會根據 **Segment Selector** 找到 GDT 中對應的 **segment descriptor**，取得 segment 的 base address、limit 和其他屬性。

在 **long mode** 下，segmentation 被停用。大多數 **segment descriptor** 的 base 和 limit 欄位將被忽略, CPU 將 address space 視為平坦的線性範圍。

Code Segment、Data Segment 和 Stack Segment 仍然存在，但只保留部分功能。CPU 仍然需要有效的 Segment Selector，但 Segment Selector 不再以傳統 segmentation 的方式執行 address translation。

**long mode** 下的 **memory translation** 幾乎完全依賴稱為 `paging` 的機制。

現在每個程式都使用稱為 virtual addresses 來運行。當 program 引用 virtual address 時，CPU 會將該位址解釋為 64 位元 linear address，並透過稱為 page tables 的多層結構進行 translation。

> 現代 x86_64 處理器支援 five-level paging，但本文將跳過它，重點介紹 four-level paging

## Translate Virtual Addr to Physical Addr by Four-level paging

在 **four-level paging** 模式下，**virtual address** 長度為 64 位元。但是，只有其中的低 48 bit 實際用於轉換為 physical address。

**Virtual address 47-0 bit**

![](https://0xax.gitbook.io/linux-insides/~gitbook/image?url=https%3A%2F%2F3490860827-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252F-M6zWfOAfArTn8XhUn_P%252Fuploads%252Fgit-blob-75990129ecb6a142f546a3e51370892b88a4dbdf%252Fearly-page-table.svg%3Falt%3Dmedia&width=400&dpr=3&quality=100&sign=42831846&sv=2)

48 bits 的 47-12 bit，以 9 bit 為一組，一共分四組。一組 9 bit 的值，代表某一層 page table 的 entry number
* 47-39 bit 代表第四層 `PML4` page table 的 entry number。
* 38-30 bit 代表第三層 `PDPT` page table 的 entry number。
* 29-21 bit 代表第二層 `PD` page table 的 entry number。
* 20-12 bit 代表第一層 `PT` page table 的 entry number。

由於 9 bit 可以表示 512 個值，因此每個 Page table 恰好包含 512 個 entry。
Page table 中的每個 entry 佔用 8 Bytes，因此單一 Page table 的 size 為 512 * 8 = 4KB。

### Step For Translating Virtual Addr to Physical Addr

當 CPU 將 **48-bit virtual address** 轉換為 **52-bit physical address** 時，它會執行以下步驟：

1. CPU 讀取 **cr3** 控制暫存器，以取得稱為 `PML4` 的 top-level page table 的 **physical address**。
2. CPU 會提取 **virtual address 的 47-39 bit**，並將它們用作 **`PML4` page table 的 index**。
3. 選定的 **`PML4` entry** 包含 next-level page table **`PDPT` 的 physical address**。
4. CPU 會提取 **virtual address 的 38–30 bit**，並將它們用作 **`PDPT` page table 的 index**。
5. 選定的 **`PDPT` entry** 包含 next-level page table **`PD` 的 physical address**。
6. CPU 會提取 **virtual address 的 29–21 bit**，並將它們用作 **`PD` page table 的 index**。
7. 選定的 **`PD` entry** 包含 next-level page table **`PT` 的 physical address**
8. CPU 會提取 **virtual address 的 20–12 bit**，並將它們用作 **`PT` page table 的 index**。
9. 選定的 **`PT` entry** 包含 **mapped physical page 的 physical address**。
10. CPU 將 **`PT` entry** 中的 **physical page address** 與 **virtual address 的 11-0 bits** 所代表的 **page offset** 組合，得到最後的 physical address。

## Page Table Entry 格式

除了包含 next-level page table 的 **physical address** 外，每個 **page table entry** 的最低 12 bit 和最高 1 bit 還包含 **flag**。

不同 paging level 的 entry 雖然具有共同的控制 bits，但部分 bits 的意義會依 entry 所處的 level，以及該 entry 是否設定 `PS` 而有所不同。

不同 level 的 page table entry 整理成以下種類
* Page Map Level 5 Entry
* Page Map Level 4 Entry
* Page Directory Pointer Table entry
* Page Directory Pointer Table entry (1 GB)
* Page Directory entry 
* Page Directory entry (2 MB)
* Page Table entry

**Page Table Entry to next-level page table** 

![](../../Images/Page-Table-Entry-to-next-level-page-table.png)

**Page Table Entry to physical memory**

![](../../Images/Page-Table-Entry-to-physical-memory.png)

### 為什麼會有 PDPTE (1 GB) 和 PDE (2 MB)

PDPTE / PDE 原本是指向下一層 table 的 entry，但 x86-64 額外允許它們直接指向一個 large page。 設定 Flag `PS`， 讓 **PDPTE / PDE 有不同的解讀**。

```
PDPTE
 ├── PS = 0 → 指向 Page Directory
 │
 └── PS = 1 → 直接描述 1 GB page
```

```
PDE
 ├── PS = 0 → 指向 Page Table
 │
 └── PS = 1 → 直接描述 2 MB page
```

### 為什麼需要 large page 設計

假設 kernel 想對應 virtual address 到 physical address 

```
physical address
0x00000000 ~ 0x3FFFFFFF
        1 GB
```

可以採用 large page 的機制，減少查閱 page table 的次數。
當 `PDPT` Entry 直接對應 1GB 的 large page。就不需要建立 low-level 的 `PD` 和 `PT`，如此一來也減少 page table 所需要占用的記憶體空間。

```
PML4
 ↓
PDPT
 ↓
1 GB page
```

相較之下，如果使用 4 KB page：

```
PML4
 ↓
PDPT
 ↓
PD
 ↓
PT
 ↓
4 KB page
```
CPU 需要繼續查找較低層的 page tables，才能找到最後對應的 physical page。

### 整理不同 paging level 的 entry 裡的 Flag 

| Bit | abbr. | Name                 | Description                          |
| --- | ----- | -------------------- | ------------------------------------ |
| 0 | `P`   | Present                  | 指示 page or page table entry 是否 valid 且存在於 memory 中。<br>如果 `P` = 0，存取對應的位址會導致 page fault。      |
| 1 | `RW`  | Read/Write               | 決定是否允許寫入操作。<br>如果 `RW` = 0, the page is read-only;<br>如果 `RW` = 1, writes are allowed (但仍需遵守權限規則).                                                                             |
| 2 | `US`  | User/Supervisor          | 控制權限等級存取權限。<br>如果 `US` = 0, 此 page 僅在 supervisor 模式下可存取。如果 `US` = 1, 也可以從 user 模式存取。       |
| 3 | `PWT` | Page-Level Write-Through | 控制 caching 策略。<br>如果 `PWT` = 0, 採用 write-back caching。<br>如果 `PWT` = 1, 採用 write-through caching。      |
| 4 | `PCD` | Page Cache Disable       | 如果 `PCD` = 1, Disables caching for the referenced page.<br>Commonly used for memory-mapped I/O regions.        |
| 5 | `A`   | Accessed                 | 當在 address translation 期間使用 page-table entry 時，CPU 會自動設定 `A` = 1。<br>Useful for page replacement decisions.                                                                  |
| 6 | `D`   | Dirty                    | 當 mapped page 進行寫入操作時，CPU 會自動設定 `D` = 1。 Indicates that the page has been modified.                    |
| 7 | `PS`  | Page Size                | 在 `PDPTE` / `PDE` 中，決定 entry 是否對應 large page（例如，2 MB 或 1 GB）。<br>而不是指向 lower-level page table。   |
| 8 | `G`   | Global                   | 告訴 CPU 在執行 `movl` to `CR3` 指令時，不要使該 page 對應的 TLB entry 失效。                                          |
| 63 | `XD`  | Execute Disable  | 如果 `XD` = 1, 阻止從 referenced page 執行指令。<br>Used to enforce executable/non-executable memory protections.        |

> 注意：上述 flags 並不是在所有 paging levels 都具有完全相同的意義；例如 `PS` 主要出現在 `PDPTE` / `PDE`，而 `D` 主要存在於最終映射 physical page 的 entry。

一個 8-byte 的 entry 如何同時包含 next-level page table 的 64-bit physical address 和 flag? 

原因是**每個 page table 都按 4kB 的邊界對齊**。因此，page table entry 的 lower 12 bits 始終為 0。這些 lower 12 bits 用於儲存 flag。

現在我們知道了 CPU 如何使用 paging 將 virtual address 轉換為 physical address，是時候看 page table 的結構了。

## Structure of page table

x86_64 的一個 page table 是 4 KB 的記憶體區域，其中有 512 個 entry，每個 entry 是 8 bytes。在採用 4-level paging mode 且每個 page table 大小為 4 KB 時，有 4 個 page table 參與 virtual address 的轉換。

| Level | Name   | Description                                              |
| ----- | ------ | -------------------------------------------------------- |
| 4     | `PML4` | The top-level page table.<br>**每個 entry 都指向一個 Page Directory Pointer Table (`PDPT`)**.                                         |
| 3     | `PDPT` | The third-level table.<br>**如果 `PS` = 0, 每個 entry 都指向一個 Page Directory (`PD`)。**<br>**如果 `PS` = 1, directly maps a 1 GB page.**                                                                             |
| 2     | `PD`   | The second-level table.<br>**如果 `PS` = 0, 每個 entry 都指向一個 Page Table (`PT`)** **<br>**如果 `PS` = 1,, directly maps a 2 MB page.** |
| 1     | `PT`   | The first-level table.<br>**每個 entry 都直接指向一個 4 KB physical memory page.**                                                     |

每個 page table 都具有相同的內部結構。不同 page table 之間唯一的區別在於**對 entry 的解讀方式**。page table 的一個 entry 長度是 64 bits。它包含兩種類型的信息：

* A physical address of either the next-level page table or a physical memory page
* A set of control flags that define access permissions and status information

## 參考資料

* [Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode 第 4 段 Set up paging](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4#set-up-paging)
* [Protected mode on x86-compatible processors](http://100.71.125.87:3000/NP0YD3GRTxSlQtAXr__Wrw)
