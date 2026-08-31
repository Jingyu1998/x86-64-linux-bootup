---
tags: early_boot, Concept
---
# Real mode on x86-compatible processors

**Real Mode 的主要特色:**
* **16-Bit Architecture**: Primarily uses **16-bit registers** and **16-bit operand/address sizes**.
* **Memory Access**: Uses `segment:offset` addressing to access a total of 1 MB of memory (2^20 bytes).
* **No Protection**: There is no memory protection or privilege levels (ring 0).
* **Backward Compatibility**: All modern x86 CPUs start in this mode.
* **Transition**: It is the default mode before switching to **protected (32-bit) or long (64-bit) mode**

## Intel 8086 

Intel 8086 是第一個支持 Real Mode 的 CPU。
Intel 8086 是 16-bit microprocessor。代表 Intel 8086 CPU 的 general-purpose register 和 intruction pointer register 都是 16-bit。
Intel 8086 CPU 具有 20 條 address bus，代表 CPU 實際上可以定址 $2^{20}$ Bytes = 1M Bytes 的記憶體。
但是 Intel 8086 CPU 的暫存器長度為 16-bit，無法使用一個暫存器表達 1M Bytes 記憶體裡每一個 Byte 的位址 

## Memory segmentation

### Problem
Intel 8086 CPU 的 暫存器長度為 16-bit，無法使用一個暫存器表達記憶體上每一個 Byte 的位址

### Purpose
讓僅有 16-bit 暫存器長度的 Intel 8086，可以產生並存取 20-bit physical address space 中的位址。

### Mechanism
採取 Memory Segmentation 機制
將記憶體視為由許多**可重疊**的 64 KB segments 所組成。

```
               +------------------+  
0xFFFFF        |                  |
               |                  |
0xF0000        +------------------+  
               |                  |
               |                  |
               |       ...        | 
               |                  |
               |                  |
               +------------------+  
0x0FFFF        |                  |
               |                  |
0x00000        +------------------+ 
```

使用兩個數值來表示記憶體位址，分別是
1. Segment selector: 用來計算 segment 的起始位址，儲存在 `cs` 暫存器 
2. Offset: 指定每一個 segment 的偏移量，儲存在 `ip` 暫存器

記憶體位址計算方式
* Physical Memory address = Base address + offset

起始位址的計算方式
* Base address = Segment selector << 4

eg: `cs:ip` 是 `0x2000:0x0010`
記憶體位址是 0x2000 << 4 + 0x0010 = 0x20010

## 參考來源

[Linux-insides Booting Chapter 第 1 篇 From bootloader to kernel 第 1 段 The Magic Power Button - What happens next?](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-1#the-magic-power-button-what-happens-next)