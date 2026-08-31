---
tags: early_boot, startup_32
---
# Stage 7: Transition to 64-bit

## Set the LME flag in EFER

### Purpose: 這段程式碼要完成什麼？

在 Extended Feature Enable Register, EFER 中設定 Long Mode Enable flag。

### Context: CPU / kernel 現在處於什麼狀態？

* 建立完成 boot page table。
* Boot Page table 一共映射 4GB 的 physical memory area。
* `CR3` 已指向 boot page table 的 Level4 page table。

### Problem: 為什麼需要這個操作？

Extended Feature Enable Register 必須設定 Long Mode Enable flag，kernel 才能啟用 Long mode。

### Implementation: 它實際怎麼完成？

EFER 是一種 model specific register, MSR。MSR 不能使用一般的 register 操作指令直接存取 。而是得採用特殊指令 `rdmsr` (Read MSR) 和 `wrmsr` (Write MSR)。

EFER 長度為 64-bit。

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L238)
```x86asm=238
	/* Enable Long mode in EFER (Extended Feature Enable Register) */
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

* `movl $MSR_EFER, %ecx`: 載入巨集 `0xc0000080` 到 `%ecx`
    * [`#define MSR_EFER		0xc0000080 /* extended feature register */`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/include/asm/msr-index.h#L10)
    * `%ecx` 指定要存取的 MSR，也就是 EFER。
* `rdmsr`:
    * 這條指令讀取 `%ecx` 中指定的 EFER。
    * 並將 64-bit 的 value 分成兩部分， higher 32 bits 存入 `%edx`，lower 32 bits 存入 `%eax`。
* `btsl $_EFER_LME, %eax`: 
    * [`#define _EFER_LME		8  /* Long mode enable */`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/include/asm/msr-index.h#L22)
    * 設定 `%eax` 的 bit 8，也就是 EFER 的 Long Mode Enable bit 為 1，並保持 EFER 其他 bit 不變。
* `wrmsr`:
    * 將 `%edx`、`%eax` 組成的 64-bit value 寫回 EFER。

### Result: 完成後 CPU / kernel 處於什麼狀態？

CPU 已完成啟用 Long mode 所需的其中一項設定。
* EFER 啟用 Long Mode Enable flag。

## 準備執行 `lret` 所需要的 stack 環境。

### Purpose: 這段程式碼要完成什麼？

這段程式碼準備執行 `lret` 所需要的 stack 環境。 

### Context: CPU / kernel 現在處於什麼狀態？

CPU 已完成啟用 Long mode 所需的其中一項設定。
* EFER 啟用 Long Mode Enable flag。

### Problem: 為什麼需要這個操作？

32-bit setup code 的最後一條指令是執行 `lret` 
執行完畢，kernel 將從 protected mode 切換到 long mode。

### Mechanism: 這段程式碼使用什麼機制?

`lret` 會 pop 出 stack 頂端的兩個值
第一個值由 `%eip` 接住
第二個值由 `%cs` 接住

```
Stack
     +------------------+  
     |  Value for eip   | top
     |  Value for cs    |
     |                  |
     +------------------+  
```

### Implementation: 它實際怎麼完成？

| 64-bit kernel code segment | value |
| -------------------------- | ----- |
| hex | `0x00af9a000000ffff` |
| Binary | 00000000 1**01**01111 10011010 00000000<br/>00000000 00000000 11111111 11111111 |

| `D`, 54th bit  | `L`, 53th bit | Meaning in code segment |
| -------------- | ------------- |----------------------- |
| 0 | 1 | 64-bit code segment |


[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L255)
```x86asm=255
	/*
	 * Setup for the jump to 64bit mode
	 *
	 * When the jump is performed we will be in long mode but
	 * in 32bit compatibility mode with EFER.LME = 1, CS.L = 0, CS.D = 1
	 * (and in turn EFER.LMA = 1).	To jump into 64bit mode we use
	 * the new gdt/idt that has __KERNEL_CS with CS.L = 1.
	 * We place all of the values on our mini stack so lret can
	 * used to perform that far jump.
	 */
	leal	rva(startup_64)(%ebp), %eax
	pushl	$__KERNEL_CS
	pushl	%eax
```

`%ebp` 儲存 startup_32 symbol 的實際 runtime address。

* `leal rva(startup_64)(%ebp), %eax`: 將 startup_64 symbol 的 runtime address 儲存於 `eax` 暫存器。
* `pushl $__KERNEL_CS`: 將 64-bit **Code Segment Selector** push 到 stack。
* `pushl %eax`: 將 startup_64 的 runtime address push 到 stack，作為 lretl 要載入的 instruction pointer。

### Result: 完成後 CPU / kernel 處於什麼狀態？

CPU 已完成啟用 Long mode 所需的其中兩項設定。
* EFER 啟用 Long Mode Enable flag。
* 準備完成 Stack 的環境。目前Stack 的頂端為: 
    * startup_64 symbol 的 physical address
    * 64-bit Code Segment Selector

## Enable Paging

### Purpose: 這段程式碼要完成什麼？
透過設定 `CR0.PG`  啟用 paging。

### Context: CPU / kernel 現在處於什麼狀態？
CPU 已完成進入 Long mode 所需的前置設定：
* `CR3` 已指向 boot page table。
* `CR4.PAE` = 1。
* `EFER.LME` = 1。

但是 `CR0.PG` 尚未設定。因此， paging 尚未正式啟用。

### Problem: 為什麼需要這個操作？

CPU 必須啟用 `CR0.PG` 才會使用 `CR3` 所指定的 page table 進行 virtual address translation。

### Implementation: 它實際怎麼完成？

kernel 載入巨集 `CR0_STATE` 到 `CR0` register
* `CR0_STATE` 定義如下：

```x86asm
#define CR0_STATE	(X86_CR0_PE | X86_CR0_MP | X86_CR0_ET | \
			 X86_CR0_NE | X86_CR0_WP | X86_CR0_AM | \
			 X86_CR0_PG)
```
 
其中 `X86_CR0_PG` 用來控制 `CR0` bit 31，是否啟用 Paging。

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L276)
```x86asm=276
	/* Enter paged protected Mode, activating Long Mode */
	movl	$CR0_STATE, %eax
	movl	%eax, %cr0
```

* `movl $CR0_STATE, %eax`：將 `CR0_STATE` 的值載入 `%eax`。
* `movl %eax, %cr0`：將 `%eax` 的值寫入 `CR0`，因此 `CR0.PG` 被設定為 1，啟用 paging。

### Result: 完成後 CPU / kernel 處於什麼狀態？

CPU 已啟用 paging

## Jump to long mode entrypoint

### Purpose: 這段程式碼要完成什麼？
kernel 執行 `lret`，切換至 long mode。也代表 kernel 進入 64-bit setup code

### Context: CPU / kernel 現在處於什麼狀態？

完成 long mode 的準備，包含
* `CR3` 已指向 boot page table。
* `CR4.PAE` = 1。
* `EFER.LME` = 1。
* `CR0.PG` = 1

### Problem: 為什麼需要這個操作？

因為 long mode 才是 x86_64 kernel 正常運行下所使用的模式。
所以要離開 protected mode 進入 long mode 繼續進行 linux 開機流程。

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L281)
```x86asm=281
	/* Jump from 32bit compatibility mode into 64bit mode. */
	lret
SYM_FUNC_END(startup_32)
```

`lret` :將 stack 頂部的值 pop 到 **Instruction Pointer**，並將下一個值 pop 到 **Code Segment register**。

## 參考來源

* [Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode 第 5 段 The transition into 64-bit mode](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4#the-transition-into-64-bit-mode)