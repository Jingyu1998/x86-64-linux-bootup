---
tags: trace_code, startup_32 
---

# Stage 3: CPU verification

## Purpose: 這段程式碼要完成什麼？
確定 CPU 支援或不支援 long mode

## Context: CPU / kernel 現在處於什麼狀態？
不確定 CPU 是否支援 long mode

## Problem: 為什麼需要這個操作？
x86-64 CPU 支援 long mode，而 kernel 需要切換至 long mode。為了切換 CPU 至 long mode，需要先確認 CPU 支援 long mode。 

## Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L133)

```x86asm=133
	/* Make sure cpu supports long mode. */
	call	verify_cpu
	testl	%eax, %eax
	jnz	.Lno_longmode
```

`verify_cpu` 函式執行 `CPUID` 指令以檢查 kernel 運作所在的 processor 的詳細資訊。
`verify_cpu` 函式會檢查多項 CPU 能力，其中關鍵的檢查為是否支援 long mode 和 支援 SSE 
`verify_cpu` 函式把回傳值放入 `%eax` 
* 回傳值 0: success
* 回傳值 1: failed

----

當回傳值是 1 時，`testl %eax, %eax` 的結果不為 0，因此 kernel 跳轉至 `no_longmode` 標籤，使用 `hlt` 的無限迴圈。

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L589)

```x86asm=589
	.text
SYM_FUNC_START_LOCAL_NOALIGN(.Lno_longmode)
	/* This isn't an x86-64 CPU, so hang intentionally, we cannot continue */
1:
	hlt
	jmp     1b
SYM_FUNC_END(.Lno_longmode)
```

## Result: 完成後 CPU / kernel 處於什麼狀態？

確定 CPU 支援 long mode，則繼續後續的 setup code
確定 CPU 不支援 long mode, 則停止執行後續的 setup code

## 參考來源
[Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode 第 3-1 段 cpu-verification](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4#cpu-verification)