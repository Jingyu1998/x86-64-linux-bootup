---
tags: early_boot, .Lrelocated
---
# Stage5: Jump on decompressed kernel entrypoint

## Purpose: 這段程式碼要完成什麼？ 

CPU 跳轉至 decompressed kernel entry point，進入 Kernel Initialization 階段。

## Context: CPU / kernel 現在處於什麼狀態？

* 已完成 kernel decompression
    * 載入 kernel image 到 memory
    * kernel image 裡需要修正的 address 已被修改為實際 load address
* decompressed kernel entry point 載入 `%rax` 

## Problem: 為什麼需要這個操作？

Kernel decompression ，接下來需要將 CPU 的控制權交給 decompressed kernel，
才能進入後續的 Kernel Initialization 階段。

## Implementation: 它實際怎麼完成？ 

[SYM_FUNC_START_LOCAL_NOALIGN(.Lrelocated) to SYM_FUNC_END(.Lrelocated)](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L479)
```x86asm=
/*
 * Jump to the decompressed kernel.
 */
	movq	%r15, %rsi
	jmp	*%rax
SYM_FUNC_END(.Lrelocated)
```

* `movq	%r15, %rsi` : 恢復 boot_params
* `jmp	*%rax` : 跳轉至 `%rax` 儲存的 decompressed kernel entry point

## Result: 完成後 CPU / kernel 處於什麼狀態？

* CPU 已離開 kernel setup 階段，進入 kernel initialization 階段。
* CPU 跳轉至 decompressed kernel entry point, 開始執行後續的 kernel initialization。

## 參考來源

* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 2 段 The last actions before the kernel decompression](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#the-last-actions-before-the-kernel-decompression)

