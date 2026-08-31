---
tags: early_boot, startup_32
---
# Stage 5: Enable PAE mode

## Purpose: 這段程式碼要完成什麼？

啟用 CPU 的 PAE (Physical Address Extension) mode，為後續 kernel 切換至 long mode 做準備。

## Context: CPU / kernel 現在處於什麼狀態？

CPU 目前仍處於 32-bit protected mode。
此時 CR4 的 PAE bit 尚未被設定，CPU 尚未啟用 PAE。

## Problem: 為什麼需要這個操作？

kernel 在切換至 long mode 前，CPU 必須先啟用 PAE。

因為 x86-64 的 long mode 使用的 paging mechanism 建立在 PAE paging 機制之上，因此 kernel 必須**先啟用 PAE**，**才能進一步建立並啟用 long mode 所需要的 page tables**。

啟用 PAE 後，CPU 會使用 PAE 格式的 paging structures，其中 **page table entries 為 64-bit**，而不是傳統 32-bit paging 使用的 **32-bit page table entries**。

## Implementation: 它實際怎麼完成？

CPU 透過修改 CR4 control register 的 PAE bit 來啟用 PAE。

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L164)

```x86asm=164
/*
 * Prepare for entering 64 bit mode
 */

	/* Enable PAE mode */
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

1. `movl %cr4, %eax`
    * 將目前 CR4 的內容讀入 `%eax`。
2. `orl $X86_CR4_PAE, %eax`: `X86_CR4_PAE` 與 `%eax` 做 Bitwise OR，並把結果寫入 `%eax`
    * `X86_CR4_PAE` 巨集展開為 0x10，也就是 bit 5 = 1, 其餘 bit = 0
    * 將 `%eax` 的 bit 5 設為 1。
    * 其他 CR4 bit 保持原本的值。
3. `movl %eax, %cr4`
    * 將修改後的值寫回 CR4。
    * 此時 CR4 的 bit 5 = 1, 也就是 PAE bit is set。CPU 因此啟用 PAE mode。

---

巨集 X86_CR4_PAE 展開

[**`/arch/x86/include/uapi/asm/processor-flags.h`**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/include/uapi/asm/processor-flags.h#L105)

```x86asm=105
#define X86_CR4_PAE_BIT		5 /* enable physical address extensions */
#define X86_CR4_PAE		_BITUL(X86_CR4_PAE_BIT)
```

`_BITUL(X86_CR4_PAE_BIT)` 展開為 1 << 5 = `0x20` 

> 巨集 `_BITUL` 展開細節，在此略過 

## Result: 完成後 CPU / kernel 處於什麼狀態？

CPU 的 CR4 的 PAE bit 已設為 1，PAE mode 已啟用。

此時
* CPU 仍處於 32-bit protected mode。
* CPU 已具備後續建立及啟用 long mode paging 所需的 PAE paging 機制。

## Reference

[ Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode 第 3-3 段 Enabling PAE mode ](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4#enabling-pae-mode)