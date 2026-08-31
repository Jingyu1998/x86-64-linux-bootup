---
tags: early_boot, startup_64
---

# Stage 6: Kernel relocation

## Purpose: 這段程式碼要完成什麼？
將 compressed kernel image 搬遷至 relocation target

## Context: CPU / kernel 現在處於什麼狀態？

* Kernel 已載入 kernel 自己定義的 gdt
* Kernel 已載入 kernel 自己定義的 idt
* Kernel 已計算出 relocation target。

kernel 已做好將 compressed kernel image 搬遷前的準備。

## Problem: 為什麼需要這個操作？

確保後續 decompression 過程中，decompressed kernel image 不會覆蓋到尚未解壓縮的 compressed kernel image。

## Implementation: 它實際怎麼完成？

下面的程式碼複製了從 `startup_32` 開始、大小為 `_bss - startup_32` bytes 的記憶體內容。

記憶體內容包含:
* 32-bit kernel setup code
* 64-bit kernel setup code
* compressed kernel code
* decompressor code

下面的程式碼使用了 `%rsi`、`%rdi` 和 `%ecx` 暫存器，這些暫存器是 x86 string operations 的標準暫存器。

由於 `std` 指令，複製操作是按**反向順序**執行的，從**較高的記憶體位址**到**較低的記憶體位址**。

`%rbx` 是 compressed kernel image 的 relocation target

[**SYM_CODE_START(startup_64) to SYM_CODE_END(startup_64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L423)
```x86asm=423
/*
 * Copy the compressed kernel to the end of our buffer
 * where decompression in place becomes safe.
 */
	leaq	(_bss-8)(%rip), %rsi
	leaq	rva(_bss-8)(%rbx), %rdi
	movl	$(_bss - startup_32), %ecx
	shrl	$3, %ecx
	std
	rep	movsq
	cld
```

* `leaq (_bss-8)(%rip), %rsi`: 載入來源位址
    * 來源位址指向 compressed kernel image 的最後 8 bytes
* `leaq	rva(_bss-8)(%rbx), %rdi`: 載入目的位址
    * 目的位址指向 relocation target 往後偏移 `rva(_bss-8)` 的位址
* `movl $(_bss - startup_32), %ecx`: 計算從 `startup_32` 開始的 image 範圍大小。
* `shrl	$3, %ecx`: 將 byte 數除以 8，計算 `rep movsq` 的 loop counter。
* `std` : 設定 Direction Flag，使 `movsq` 每次複製後都會**遞減** `%esi` 和 `%edi`。
* `rep movsq`: CPU 重複執行 `%ecx` 次，每次將 8 bytes 從 `%rsi` 指向的記憶體位址**複製**到 `%rdi`指向的記憶體位址。

## Result: 完成後 CPU / kernel 處於什麼狀態？

kernel 已將 compressed kernel image 搬遷至 relocation

## 參考來源

[Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 1-6 段 Kernel relocation](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#kernel-relocation)
