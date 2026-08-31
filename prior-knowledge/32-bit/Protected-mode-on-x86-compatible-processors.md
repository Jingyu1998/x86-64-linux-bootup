---
tags: early_boot, Concept
---
# Protected mode on x86-compatible processors

在 protected mode 下, 存取記憶體位置，需要結合使用兩個CPU暫存器
* A segment register - `cs`, `ds`, `ss` and `es` which defines **segment selector**.
* A general purpose register which specifies **offset** within the segment.

## Main motivation to protected mode

Main motivation for switching from real mode to protected mode is its memory addressing limitation.

## Intel 80286

Intel 80286 是第一個支持 Protected Mode 的 CPU。
Intel 80286 CPU 具有 24 條 address bus。Address width = 24-bit。代表 CPU 實際上可以定址 $2^{24}$ Bytes = 16M Bytes 的記憶體
Intel 80286 使用 16-bit general purpose register

## Intel 80386, i386

i386 CPU 具有 32 條 address bus。Address width = 32-bit。代表 CPU 實際上可以定址 $2^{32}$ Bytes = 4G Bytes 的記憶體
i386 使用 32-bit general purpose register

## Two Memory management mechanism
Memory management in protected mode is divided into two, mostly independent mechanisms:

* Segmentation
* Paging

For now, our attention stays on segmentation. 

## Memory segmentation in protected mode

Protected mode 捨棄原先 real mode 採用的固定 64KB segments。
Protected mode 改成使用 Segment Descriptor 這個特殊資料結構來定義 segment。

Segment Descriptor
* Segment Descriptor 指定了 memory segment 的屬性
* Segment Descriptor 儲存在一個名為 Global Descriptor Table（GDT）的特殊結構中。
* 一個 Segment Descriptor 為 64-bits

GDT
* GDT 儲存 Segment Descriptor
* 當 CPU 需要尋找 physical memory address 時，它會查閱 GDT。
* GDT 本身只是一塊記憶體。
* GDT 的位址儲存在 `gdtr` 的這個特殊 CPU 暫存器中。

GDTR
* GDTR 是一個 48 位元暫存器, 由兩部分組成。
* The **Limit** of the Global Descriptor Table, 2 Bytes
* The **base address** of the Global Descriptor Table, 4 Bytes


CPU 提供的專用指令: 將 GDT 位址載入到 GDTR 暫存器中
`lgdt gdt`

![](http://100.71.125.87:3000/uploads/upload_ce0793617882af6acd9694b48667043d.png)

將 GDT 位址載入到 GDTR 暫存器的圖示
![](http://100.71.125.87:3000/uploads/upload_223d78faf279194d5347e54523c67ffc.png)

## Segment Descriptor 

![](https://0xax.gitbook.io/linux-insides/~gitbook/image?url=https%3A%2F%2F3490860827-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252F-M6zWfOAfArTn8XhUn_P%252Fuploads%252Fgit-blob-ba9e458330319429977cdc01cd097c43934d74c8%252Fsegment-descriptor.svg%3Falt%3Dmedia&width=400&dpr=3&quality=100&sign=2ea5b294&sv=2)

將 Segment Descriptor Field 分為

1. 決定 segment 邊界的 Segment Descriptor Field
    * Limit, 分段邊界
    * G: 表示 Limit 的單位
2. 決定 segment 起始位址 的 Segment Descriptor Field
    * BASE, 起始位址
3. 決定 segment 種類和型態 的 Segment Descriptor Field
    * S 
    * Type
4. 剩餘的 Segment Descriptor Field

### 決定 segment 邊界的 Segment Descriptor Field

Limit, 分段邊界: 
* 表示一個 segment 的邊界
* 長度為 20-bit。
* 分散存放在 Segment Descriptor 的 0-15 bit 和 48-51 bit

G: 表示 Limit 的單位。存放在 Segment Descriptor 的 55th bit
* 當 G 為 0 時，Segment 的 size 是以 Byte 為單位。所以最大的 Segment size 是  1 * $2^{20}$ Bytes = 1M Bytes。
* 當 G 為 1 時，Segment 的 size 是以 4KB 為單位。所以最大的 Segment size 是 4K * $2^{20}$ Bytes = 4G Bytes。

### 決定 segment 起始位址 的 Segment Descriptor Field

BASE, 起始位址:
* 表示一個 segment 的起始位址
* 長度為 32-bit。40-43 bit
* 分散存放在 Segment Descriptor 的 16-32 bit 和 32-39 bit 和 56-63 bit

### 決定 segment 種類和型態 的 Segment Descriptor Field

`S` - distinguishes system segments from code and data segments.

`Type` - describes the type of a memory segment.
* 長度為 4-bit。存放在 Segment Descriptor 的 40-43 bit

`S`: 44th bit
* S = 0 時表示 segment 是一個 system segment
* S = 1 時則表示 segment 是一個 code / data segment。 

在 S = 1 的情況下
Hightest bit of `Type` field: 43rd bit
* 43rd bit = 0, 表示 segment 是一個 data segment 
* 43rd bit = 1, 表示 segment 是一個 code segment

| S, 44th bit | Hightest bit of `Type` field, 43rd bit | Meaning |
| ----------- | -------------------------------------- | ------- |
| 0 | x | system segment |
| 1 | 0 | data segment |
| 1 | 1 | code segment |

---

**data segment** 的 Lower 3 bit of `Type` field: 42-40 bit
* `Accessed`, 40 bit: OS 上一次清除這個 bit 之後，CPU 是否使用這個 data segment。
* `Write-Enable`, 41 bit: determines whether a data segment is writable or read-only.
* `Expansion-Direction`, 42 bit: determines whether addresses decreasing from the base address or not

| `Accessed`, 40 bit | Meaning |
| ------------------ | ------- |
| 0 | OS 上一次清除這個 bit 之後，CPU **沒有使用**這個 data segment |
| 1 | OS 上一次清除這個 bit 之後，CPU **有使用**這個 data segment|

| `Write-Enable`, 41 bit | Meaning |
| ---------------------- | ------- |
| 0 | data segment 唯讀 |
| 1 | data segment 可讀寫 |

| `Expansion-Direction`, 42 bit | Meaning |
| ----------------------------- | ------- |
| 0 | data segment 的有效 offset 範圍由**低位址**往**高位址**方向延伸 |
| 1 | data segment 的有效 offset 範圍由**高位址**往**低位址**方向延伸|

---

**code segment** 的 Lower 3 bit of `Type` field: 42-40 bit
* `Accessed`, 40 bit: OS 上一次清除這個 bit 之後，CPU 是否使用這個 code segment。
* `Read-Enable`, 41 bit: determines whether a code segment is readable or execute-only.
* `Conforming`, 42 bit: 決定這個 Code Segment 是否允許較低權限的程式碼直接進入執行。但是較低權限的程式碼仍保留原本的權限。

| `Accessed`, 40 bit | Meaning |
| ------------------ | ------- |
| 0 | OS 上一次清除這個 bit 之後，CPU **沒有使用**這個 code segment |
| 1 | OS 上一次清除這個 bit 之後，CPU **有使用**這個 code segment|

| `Read-Enable`, 41 bit | Meaning |
| ---------------------- | ------- |
| 0 | code segment 唯執行 |
| 1 | code segment 可讀可執行 |

| `Conforming`, 42 bit | Meaning |
| ----------------------------- | ------- |
| 0 | 這個 Code Segment **不允許**較低權限的程式碼直接進入執行 |
| 1 | 這個 Code Segment **允許**較低權限的程式碼直接進入執行 |

### 剩餘的 Segment Descriptor Field

* `DPL`, 45-46 bit - provides information about the **privilege level** of a segment. It can be a value from 0 to 3, where 0 is the level with the highest privileges.
* `P`, 47th bit - 表示這個 segment 是否為 present，可供 CPU 使用。
* `AVL`, 52th bit - available and reserved bits. It is ignored by the Linux kernel.
* `L`, 53th bit - indicates whether a **code segment** contains 64-bit code.
* `D/B`, 54th bit: 在不同segment type 下有不同的意義
 
**code segment 的 D flag**

| `D`, 54th bit    | Meaning in code segment |
| ---------------- | ----------------------- |
| 0 | 16-bit code segment |
| 1 | 32-bit code segment |

**stack segment 的 B flag**

| `B`, 54th bit    | Meaning in stack segment |
| -----------------| ------------------------ |
| 0 | 16-bit stack segment |
| 1 | 32-bit stack segment |

**expand-down data segment 的 B flag**

| `B`, 54th bit    | Meaning in expand-down data segment |
| ---------------- | ----------------------------------- |
| 0 | 16-bit data segment, upper bound is `0xFFFF` |
| 1 | 32-bit data segment, upper bound is `0xFFFFFFFF` |

## 比較: 一般 和 Expansion-down 的 data segmetion

一般的 data segmetion
* offset 的有效範圍是 0 <= offset <= Limit
```
        +------------------+  
Limit   |                  |
        |                  |
        |  segment         |
        |                  |
0       +------------------+ 
```

Expansion-down 的 data segmetion
* offset 的有效範圍是 Limit < offset <= Upper Bound
* B = 0 時，Upper Bound = 0xFFFF
* B = 1 時，Upper Bound = 0xFFFFFFFF

```
             +------------------+  
Upperbound   |                  |
             |                  |
             |  valid           |
             |                  |
             +------------------+  
Limit        |                  |
             |                  |
             |  invalid         |
             |                  |
0            +------------------+ 
```

## Segment selector

In protected mode, CPU 使用 segment selector 來引用 segment descriptor

![](https://0xax.gitbook.io/linux-insides/~gitbook/image?url=https%3A%2F%2F3490860827-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252F-M6zWfOAfArTn8XhUn_P%252Fuploads%252Fgit-blob-65126cc3d2ecb9c00c74723d05f15705f852e6e7%252Fsegment-selector.svg%3Falt%3Dmedia&width=400&dpr=3&quality=100&sign=4137f34d&sv=2)

segment selector
* 長度為 16-bit。

The meaning of the segment selector fields is:
* `Index` - the entry number of the segment descriptor in the descriptor table.
* `TI` - indicates where to search for the segment descriptor
    * If the value of the bit is 0, a segment descriptor will be searched in the Global Descriptor Table.
    * If the value of this bit is 1, a segment descriptor will be searched in the Local Descriptor Table.
* `RPL` - the privilege level requested by the selector.

## CPU 如何計算正確的 physical address

當在 protected mode 運行的程式需要參考記憶體時，CPU 採取以下步驟計算出正確的 physical address。

1. 將 Segment selector 載入到其中一個 Segment register 中。
2. CPU 會根據 Segment selector 提供的索引值，在 GDT 中尋找對應的 segment descriptor。如果找到 segment descriptor，則會將其載入到該 segment register 的一個特殊隱藏部分。
3. physical address 將是 segment descriptor 的 base address 加上 offset from the **instruction pointer** or **memory location** referenced within an executed instruction.

## 參考來源
* [Linux-insides Booting Chapter 第 2 篇 First steps in the kernel setup code 第 1 段 Protected mode](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-2#protected-mode)
* [將 GDT 位址載入到 GDTR 暫存器](https://i-blog.csdnimg.cn/blog_migrate/2ede905fe12cfedd8ef73a31a17461db.png#pic_center)
* [GDTR 暫存器](https://wiki.osdev.org/Global_Descriptor_Table#GDTR)
* [分段架構](https://www.csie.ntu.edu.tw/~wcchen/asm98/asm/proj/b85506061/chap2/segment.html)