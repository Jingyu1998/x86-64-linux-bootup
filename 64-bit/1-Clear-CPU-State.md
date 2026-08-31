---
tags: early_boot, startup_64
---

# Stage 1: Clear CPU State

## Purpose: 這段程式碼要完成什麼？
將 Direction Flag 和 Interrupt flag 清為 0

## Context: CPU / kernel 現在處於什麼狀態？

進入 64-bit kernel Setup code 的方式
* 除了執行 32-bit kernel Setup code 的最後一條指令 `lret` ，接著跳轉到 64-bit kernel Setup code。
* 也可以是由 bootloader 自行將 CPU 切換到 64 bit long mode。並且將 CPU 控制權交給 kernel 從 64-bit kernel Setup code 開始執行。

若是採取第二種方式進入 64-bit kernel Setup code，kernel 無法確定 bootloader 是否有將 Direction Flag 和 Interrupt flag 清為 0。

| Direction Flag | Interrupt flag |
| -------------- | -------------- |
| 不確定是否為0 | 不確定是否為0 |

## Problem: 為什麼需要這個操作？

- 確保後續程式碼需要用到 string operations 時，會對 index register, eg: `esi`、`edi` 採取 **auto-increment** 
- 不希望 CPU 在 kernel decompression 過程中收到中斷而受到干擾。

## Implementation: 它實際怎麼完成？

[**SYM_CODE_START(startup_64) to SYM_CODE_END(startup_64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L298)
```x86asm=298
	cld
	cli
```

## Result: 完成後 CPU / kernel 處於什麼狀態？

| Direction Flag | Interrupt flag |
| -------------- | -------------- |
| 0 | 0 |

不管 kernel 採用什麼方式進入 64-bit kernel Setup code，現在都已確保 Direction Flag 和 Interrupt Flag 為 0。

## 參考來源 

[Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 1-1 段 Disable the interrupts](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#disabling-the-interrupts)
