---
tags: early_boot, .Lrelocated
---
# Stage 4: Kernel decompression

## Purpose: 這段程式碼要完成什麼？ 

* 將 compressed kernel image 解壓縮，得到 ELF executable file `vmlinux`。
* 將 ELF 的 `PT_LOAD` segments **載入** physical memory。
* 將已載入的 kernel image 裡**需要修正的 address**, 依據實際 load address 與 link-time address 之間的 **relocation delta** 進行修正。 


## Context: CPU / kernel 現在處於什麼狀態？

* Kernel 已完成 relocation。
* CPU 已跳轉至 relocated kernel code。
* Kernel 已載入 Stage2 IDT。
* Kernel 已建立並啟用新的 identity mapping page table。
* Kernel 尚未解壓縮 compressed kernel image。

接下來將呼叫 `extract_kernel()` 執行 kernel decompression。

## Problem: 為什麼需要這個操作？

目前記憶體中的 kernel 仍然是 compressed kernel image，尚未成為可以直接執行的完整 kernel image。

## Implementation: 它實際怎麼完成？ 

[SYM_FUNC_START_LOCAL_NOALIGN(.Lrelocated) to SYM_FUNC_END(.Lrelocated)](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L471)
```x86asm=471
/*
 * Do the extraction, and jump to the new kernel..
 */
	/* pass struct boot_params pointer and output target address */
	movq	%r15, %rdi
	movq	%rbp, %rsi
	call	extract_kernel		/* returns kernel entry point in %rax */
```

`%r15` : 存放 `boot_params` 的起始位址
`%rbp` : 存放 kernel decompression 的 output physical address。

* `movq %r15, %rdi`: 
    * 將 `boot_params` 的起始位址作為 `extract_kernel()` 的第一個引數。
* `movq %rbp, %rsi`:
    * 將 解壓縮 kernel 的起始位址作為 `extract_kernel()` 的第二個引數。
* `call extract_kernel`:
    * 呼叫 `extract_kernel()`
    * 函式完成後，`%rax` 返回 decompressed kernel entry point。

大致閱讀 function `extract_kernel` 的實作。並將函式實作分成以下主題來探討
* 初始化 decompressor 的 heap
* 選擇 kernel output location
* 檢查 kernel output location
* 呼叫 `decompress_kernel` 函式，函式實作分成以下主題來探討
    * Decompress kernel
    * Parse kernel ELF
    * Handle relocations

## 初始化 decompressor 的 heap

`extract_kernel()` 會設定 decompressor 使用的 heap：

[function `extract_kernel`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/misc.c#L469) 
```c=469
	free_mem_ptr     = heap;	/* Heap */
	free_mem_end_ptr = heap + BOOT_HEAP_SIZE;
```

* `free_mem_ptr`
    * 指向 decompressor 可使用的 heap 起始位置。
* `free_mem_end_ptr`
    * 指向 heap 的結束位置。

decompressor 在執行 decompression 時會大量使用這個 heap。

## 選擇 kernel output location

`extract_kernel()` 會呼叫 `choose_random_location()`。

* 如果啟用 KASLR，這個函式會選擇 kernel 解壓縮後的 output location。
* 如果停用 KASLR, 解壓縮 compressed kernel image 的起始位址不會被改變，直接使用傳入給 `extract_kernel()` 的第二個引數。

## 檢查 kernel output location

在開始 decompression 前，kernel 會檢查 destination address 是否符合要求。
destination address 是解壓縮 compressed kernel image 的起始位址。

[function `extract_kernel`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/misc.c#L506)

```c=506
	/* Validate memory location choices. */
	if ((unsigned long)output & (MIN_KERNEL_ALIGN - 1))
		error("Destination physical address inappropriately aligned");
	if (virt_addr & (MIN_KERNEL_ALIGN - 1))
		error("Destination virtual address inappropriately aligned");
#ifdef CONFIG_X86_64
	if (heap > 0x3fffffffffffUL)
		error("Destination address too large");
	if (virt_addr + needed_size > KERNEL_IMAGE_SIZE)
		error("Destination virtual address is beyond the kernel mapping area");
#else
...
#endif
#ifndef CONFIG_RELOCATABLE
	if (virt_addr != LOAD_PHYSICAL_ADDR)
		error("Destination virtual address changed when not relocatable");
#endif
```

destination physical address 是否符合 `MIN_KERNEL_ALIGN` 對齊要求。
* `if ((unsigned long)output & (MIN_KERNEL_ALIGN - 1))`:<br>**If 條件成立**, 代表 destination physical address **沒有對齊** `MIN_KERNEL_ALIGN`。 

destination virtual address 是否符合 `MIN_KERNEL_ALIGN` 對齊要求。
* `if (virt_addr & (MIN_KERNEL_ALIGN - 1))`:<br>**If 條件成立**, 代表destination virtual address **沒有對齊** `MIN_KERNEL_ALIGN`。 

heap 是否超出 decompressor **在此階段**允許使用的 address range (`0x3fffffffffffUL`)。
* `if (heap > 0x3fffffffffffUL)`: <br>**If 條件成立**, 代表 heap 超出 decompressor **在此階段**允許使用的 address range

從 virt_addr 開始、大小為 needed_size 的整個 kernel 所需的範圍，是否仍然位於允許的 kernel mapping area 內。
* `if (virt_addr + needed_size > KERNEL_IMAGE_SIZE)`: <br>**If 條件成立**, kernel 所需的範圍已經超出允許的 kernel mapping area。

kernel **不能** relocation 時，destination virtual address 是否仍為預期的 `LOAD_PHYSICAL_ADDR`。
* `if (virt_addr != LOAD_PHYSICAL_ADDR)`

## 呼叫 decompress_kernel

[ function `extract_kernel` ](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/misc.c#L532)
```c=532
	entry_offset = decompress_kernel(output, virt_addr, error);
```

函式實作分成以下主題來探討
* Decompress kernel
* Parse kernel ELF
* Handle relocations

### Decompress the kernel

透過 `__decompress()` 使用 kernel build 時選擇的 compression algorithm 解壓縮 kernel。

例如：
* GZIP → `decompress_inflate.c`
* BZIP2 → `decompress_bunzip2.c`
* LZMA → `decompress_unlzma.c`
* XZ → `decompress_unxz.c`
* LZO → `decompress_unlzo.c`
* LZ4 → `decompress_unlz4.c`
* ZSTD → `decompress_unzstd.c`

解壓縮 compressed kernel image 得到的 kernel binary, `vmlinux`

### Parse kernel ELF binary

`vmlinux` 是 ELF executable file。
因此，解壓縮後，我們得到的不僅僅是一段 code，而是一個包含 header、program segment、debug symbol 和其他資訊的 ELF 檔案。

`parse_elf` 函數代表 ELF 載入器。

`parse_elf()` 會：
* 讀取 kernel ELF 的 program headers。
* 找出需要載入的 `PT_LOAD` segments。
* 將各個 `PT_LOAD` segment **載入**到其應該所在的 physical memory。

因此完成後，kernel 的 code、data 等 segments 已經被放置到 physical memory 裡選定的 load address。
然而，這可能不足以使 kernel 完全可運行。

### Handle relocations

`vmlinux` 最初是按照**特定的基底位址** link。
如果啟用 KASLR ，kernel 可以**載入**到不同的 physical address 和 virtual address。

因此，`vmlinux` 中嵌入的任何 absolute address 仍將反映 link-time address，而不是實際 load address。

為了解決這個問題，`vmlinux` 包含一個 **relocation table**，用於**記錄需要修正的 address**。

`handle_relocations()` 會：

* 讀取 relocation table 記錄的 address。
* 計算 relocation delta。<br>**relocation delta** 是 `vmlinux` 的實際 load address $-$ `vmlinux` link-time address
* 將需要修正的 address 加上 relocation delta。

使 kernel image 中的 absolute address 符合實際的 load address。

## Result: 完成後 CPU / kernel 處於什麼狀態？

* compressed kernel image 已經解壓縮。
* ELF 中的 `PT_LOAD` segments 已載入至正確的 physical memory。
* kernel image 裡需要修正的 address 已被修改為實際 load address
* `extract_kernel()` 在 `%rax` 返回 decompressed kernel entry point。

## 參考來源

* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 2 段 The last actions before the kernel decompression](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#the-last-actions-before-the-kernel-decompression)
* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 3 段 Kernel decompression](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#kernel-decompression)
* [ELF, Executable and Linking Format](../prior-knowledge/Relocated/ELF-Executable-and-Linking-Format.md)