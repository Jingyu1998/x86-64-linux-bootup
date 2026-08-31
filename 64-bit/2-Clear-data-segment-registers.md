---
tags: early_boot, startup_64
---
# Stage 2: Clear data segment registers

## Purpose: 這段程式碼要完成什麼？

將 data segment register 清為 0

## Context: CPU / kernel 現在處於什麼狀態？

進入 64-bit kernel Setup code 的方式
* 除了執行 32-bit kernel Setup code 的最後一條指令 `lret` ，接著跳轉到 64-bit kernel Setup code。
* 也可以是由 bootloader 自行將 CPU 切換到 64 bit long mode。並且將 CPU 控制權交給 kernel 從 64-bit kernel Setup code 開始執行。

若是採取第一種方式進入 64-bit kernel Setup code，則 data segment register:
* `%ds`
* `%es`
* `%ss`
* `%fs`
* `%gs`

會殘留 protected mode 載入的 segment selector。

## Problem: 為什麼需要這個操作？

Long mode 下，大部分 data segmentation 被弱化，不需要依賴 protected mode 遺留的 segment selector 進行一般的 virtual address 計算。

因此需要將這些 segment registers 統一清為 0，避免後續程式碼依賴進入 long mode 前所留下的 segment state。

## Implementation: 它實際怎麼完成？

[**SYM_CODE_START(startup_64) to SYM_CODE_END(startup_64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L301)
```x86asm=301
	/* Setup data segments. */
	xorl	%eax, %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
	movl	%eax, %fs
	movl	%eax, %gs
```

## Result: 完成後 CPU / kernel 處於什麼狀態？

data segment register 已清除可能來自 protected mode 載入的 data segment selector。

## 參考來源 

[Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 1-2 段 Unification of the segment registers](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#unification-of-the-segment-registers)
