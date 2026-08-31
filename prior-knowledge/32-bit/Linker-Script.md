---
tags: early_boot, Concept
---

# Linker Script

Linker Script 是用來**描述** 「Linker 應該如何**安排最終 binary image 或 executable 的 memory layout**」 的**腳本**。

## Purpose

* 定義 linker 產生最終 binary / executable 時的 **memory layout**。
* 指定不同 **input sections** 應該如何被組合成 **output sections**。
* 指定各個 section 的排列順序、位址，以及必要的 linker symbols。

## Problem

Compiler 編譯各個 source file 時，會分別產生 object file。

例如：
```
foo.o
├── .text    # 程式碼
├── .rodata  # 唯讀資料
└── .data    # 已初始化資料

bar.o
├── .text    # 程式碼
├── .rodata  # 唯讀資料
└── .data    # 已初始化資料
```

這些 object file 只知道自己有哪些 sections，**並沒有完整描述最終 binary 中所有 sections 應該如何排列與放置**。

因此 linker 需要知道：

* 哪些 input sections 要合併
* 合併後的 output sections 如何排列
* 每個 section 從哪個 address 開始
* 是否需要建立代表特定位址的 symbols

Linker Script 就是用來描述這些規則。

## Mechanism

Linker Script 透過定義 **output sections**，將不同 object files 中的 **input sections** 組合成最終的 binary layout。

基本關係可以理解成：

```
Object files
     │
     ├── .text
     ├── .rodata
     └── .data
          │
          ▼
    Linker Script
          │
          │ 決定
          │ ・section 如何合併
          │ ・section 排列順序
          │ ・section 位址
          │ ・symbols
          ▼
   Final executable / image
```

例如：

```
foo.o              bar.o
  │                  │
  └── .text ─────────┘
           │
           ↓
      output .text

foo.o              bar.o
  │                  │
  └── .rodata ───────┘
           │
           ↓
      output .rodata
```

## Implementation

使用 Linker Script 描述 linker 如何建立最終的 memory layout。

例如：
```
SECTIONS
{
    . = 0x1000;

    .text : {
        *(.text)
    }

    .rodata : {
        *(.rodata)
    }

    .data : {
        *(.data)
    }
}
```

其中
* `SECTIONS`：定義最終 output image 中各個 section 的配置。
* `. = 0x1000`：設定 linker 的 **location counter**，表示接下來從 `0x1000` 開始配置。
* `.text`：建立 output section。
* `*(.text)`：將各個 input object file 中的 `.text` section 放入這個 output section。
* `*(.rodata)`: 將各個 input object file 中的 `.rodata` section 放入這個 output section。
* `*(.data)`: 將各個 input object file 中的 `.data` section 放入這個 output section。