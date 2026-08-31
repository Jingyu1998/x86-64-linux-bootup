---
tags: early_boot, Concept
---

# BSS Section 

## Static Storage Duration, 靜態儲存週期

在**函式外的全域變數**或在**函式內但是刻意以static關鍵字修飾的變數**會是屬於static storage duration。他的生命週期(lifetime)是從程式開始執行的時候開始，程式結束之後才會被釋放。整個程式運行時會佔用固定的記憶體空間。

```c
int g_variable1;     /* Static Storage Duration */
int g_variable2 = 5; /* Static Storage Duration */
int g_array[10];     /* Static Storage Duration */

int main() {
    static int local_static_variable; /* Static Storage Duration */
    return EXIT_SUCCESS;
}
```

static storage duration 的變數會有以下特性：
* 如果宣告時有被初始化，其值會等於初始值。
* 如果宣告時沒有被初始化，且其型別為算術型別(arithmetic type)，其值會被初始化為 0。
* 如果宣告時沒有被初始化，且其型別為指標型別，其值會被初始化為NULL。

所以變數的初值如下 
* `g_variable1 = 0`
* `local_static_variable = 0`
* `g_variable2 = 5`
* `g_array[0] = 0`
* `g_array[1] = 0`
* ...
* `g_array[9] = 0`


## BSS Section in Compressed kernel image

Block starting symbol（縮寫為 .bss 或 bss）是 object file、executable 或 assembly code 中包含 static variables 的部分，這些變數已 declared 但尚未 assigned 。它通常被稱為“bss section” 或 “bss segment”。

### Purpose

* 提供儲存未初始化的 static variables 的 memory area。
* 確保這些變數在 C code 開始執行時，其初始值符合 C 語言規則，即為 0。

### Problem

* C 語言規定，具有 static storage duration 且沒有明確 initializer 的變數，初始值必須為 0。
* 如果直接將這些變數的 0 值完整儲存在 kernel image 中，會增加 image 的大小。

### Implementation

* Compiler / linker 將未初始化的 global 和 static variables 配置到 `.bss` section。
*  在 Linux compressed kernel image 中，Linker script 使用 `_bss` 和 `_ebss` 標記 `.bss` 的起始與結束位置。

## 參考來源

[.bss](https://en.wikipedia.org/wiki/.bss)
[C/C++的初始化規則與變數的儲存週期](https://hackmd.io/@HsuChiChen/memory-layout-in-c)