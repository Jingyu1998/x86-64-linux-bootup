---
tags: early_boot, startup_64
---
# Stage 8: Jump on the relocated code

## Purpose: 這段程式碼要完成什麼？

跳入 kernel relocation 後的 compressed kernel image 裡的 relocated code

## Context: CPU / kernel 現在處於什麼狀態？

目前 CPU 仍在 bootloader 載入的原始 compressed kernel image 中執行。

此時 compressed kernel image 已經完成 relocation，但 CPU 尚未跳到 relocation 後的 code。

## Problem: 為什麼需要這個操作？

後續執行 decompression，應該要是從 kernel relocation 後的 compressed kernel image 裡的 relocated code 接續執行。

## Implementation: 它實際怎麼完成？

[**SYM_CODE_START(startup_64) to SYM_CODE_END(startup_64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L423)
```x86asm=423
/*
 * Jump to the relocated address.
 */
	leaq	rva(.Lrelocated)(%rbx), %rax
	jmp	*%rax
SYM_CODE_END(startup_64)
```

`%rbx` 是 compressed kernel image 的 relocation target

* `leaq rva(.Lrelocated)(%rbx), %rax`: 將 relocation 後的 Symbol `.Lrelocated`  的 runtime address 載入 `%rax` 
* `jmp *%rax`: 將 %rax 的值載入 RIP，跳至 relocated code entry point 執行。

## Result: 完成後 CPU / kernel 處於什麼狀態？

kernel 已進入 relocated code entry point。

## 參考來源

[Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 1-6 段 Kernel relocation](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#kernel-relocation)