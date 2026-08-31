---
tags: early_boot, .Lrelocated
---
# Stage 1: Initialize the BSS section

## Purpose: 這段程式碼要完成什麼？

將 Symbol `_bss` 到 Symbol `_ebss` 所涵蓋的記憶體空間初始化為 0

## Context: CPU / kernel 現在處於什麼狀態？

* Kernel 已完成 relocation。
* CPU 已跳轉至 relocated kernel code。

## Problem: 為什麼需要這個操作？

BSS section 儲存未初始化的 global 和 static variables。
在 C 語言中，具有 static storage duration 且沒有明確 initializer 的變數，初始值必須為 0。

但 kernel 不能假設目前 bss section 已被初始化為0。

## Implementation: 它實際怎麼完成？

[SYM_FUNC_START_LOCAL_NOALIGN(.Lrelocated) to SYM_FUNC_END(.Lrelocated)](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L455)

```x86asm=455
/*
 * Clear BSS (stack is currently empty)
 */
	xorl	%eax, %eax
	leaq    _bss(%rip), %rdi
	leaq    _ebss(%rip), %rcx
	subq	%rdi, %rcx
	shrq	$3, %rcx
	rep	stosq
```

* `xorl %eax, %eax`: `%eax` 設為 0
* `leaq _bss(%rip), %rdi`: 將 BSS section 起始位址 `_bss` 放入 `%rdi`。
* `leaq _ebss(%rip), %rcx`: 將 BSS section 結束位址 `_ebss` 放入 `%rcx`。
* `subq %rdi, %rcx`: 計算 BSS section 的 size
* `shrq	$3, %rcx`: 將 byte 數除以 8，計算 `rep stosq` 的 loop counter。
* `rep stosq`: 將 BSS section 初始化為 0
    *  重複執行 `stosq` 指令，直到 `%rcx` 變為 0。
    *  執行一次 `stosq` 指令，將 `%rax` 的值寫入到 `%rdi` 所指向的記憶體位址。
    *  每次執行後 `%rdi` 自動增加 8。
    *  每次執行後 `%rcx` 自動減少 1。


## Result: 完成後 CPU / kernel 處於什麼狀態？

* kernel 確保後續包含 decompressor code 的程式碼所需要的 BSS Section 都已初始化為 0

## 參考來源

* [BSS Section](http://100.71.125.87:3000/oXgeUQp7Rq-sGKL5zUjwiA)
* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 2 段 The last actions before the kernel decompression](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#the-last-actions-before-the-kernel-decompression)

