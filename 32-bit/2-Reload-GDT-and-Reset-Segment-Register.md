---
tags: early_boot, startup_32
---
# Stage 2: Reload GDT and Reset Segment Register

## Reload GDT

### Purpose: 這段程式碼要完成什麼？
將 GDT 重新載入 GDTR

### Context: CPU / kernel 現在處於什麼狀態？
kernel 並未使用在 protected mode 下新定義的 GDT 

### Problem: 為什麼需要這個操作？
為了完成切換至 long mode 前的準備，kernel 必須使用在 protected mode 下新定義的 GDT 。

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L107)
```x86asm=107
	leal	rva(gdt)(%ebp), %eax
	movl	%eax, 2(%eax)
	lgdt	(%eax)
```

The `ebp` register contains the physical address of the `startup_32` symbol.

* `leal	rva(gdt)(%ebp), %eax`: 計算 Symbol `gdt` 的 physical memory address。

![](http://100.71.125.87:3000/uploads/upload_ce0793617882af6acd9694b48667043d.png)

* `movl %eax, 2(%eax)`: Symbol `gdt` 實際上指向 GDT Descriptor（一個 6-byte的結構）。這個指令將 **`%eax` 的值**寫入 **`%eax + 2` 的記憶體位址**，此時 GDT Descriptor 的 base address 欄位被設為 gdt 的 physical address。
* `lgdt (%eax)`: 這個指令載入 GDT Descriptor，更新 GDTR，使 GDTR 指向新的 GDT。


### Result: 完成後 CPU / kernel 處於什麼狀態？
kernel 已使用在 protected mode 下新定義的 GDT 

## protected mode 的 GDT 結構

[The new Global Descriptor Table looks like this:](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L606)

```x86asm
SYM_DATA_START_LOCAL(gdt)
	.word	gdt_end - gdt - 1   /*  GDTR  Limit */
	.long	0                   /*  32-bit GDTR Base */         
	.word	0                   
	.quad	0x00cf9a000000ffff	/* __KERNEL32_CS */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
SYM_DATA_END_LABEL(gdt, SYM_L_LOCAL, gdt_end)
```

new GDT 的結構，會在開頭先定義 GDT Descriptor
* GDT Descriptor Limit
* 32-bit GDT Descriptor Base


The new **Global Descriptor table contains five descriptors**:
* 32-bit kernel code segment
* 64-bit kernel code segment
* 32-bit kernel data segment
* Task state descriptor
* Second task state descriptor

----

| 32-bit kernel code segment | value |
| -------------------------- | ----- |
| hex | `0x00cf9a000000ffff` |
| Binary | 00000000 1**10**01111 10011010 00000000<br/>00000000 00000000 11111111 11111111 |

| `D`, 54th bit  | `L`, 53th bit | Meaning in code segment |
| -------------- | ------------- |----------------------- |
| 1 | 0 | 32-bit code segment |

----

| 64-bit kernel code segment | value |
| -------------------------- | ----- |
| hex | `0x00af9a000000ffff` |
| Binary | 00000000 1**01**01111 10011010 00000000<br/>00000000 00000000 11111111 11111111 |

| `D`, 54th bit  | `L`, 53th bit | Meaning in code segment |
| -------------- | ------------- |----------------------- |
| 0 | 1 | 64-bit code segment |

## Reset Segment Register excluding Code Segment Register

### Purpose: 這段程式碼要完成什麼？
Segment register 設為 `__BOOT_DS = 0x18`

### Context: CPU / kernel 現在處於什麼狀態？
Segment register 可能還是讀取 real mode 的 Segment selector

### Problem: 為什麼需要這個操作？
如果這些 Segment selector 指向新的 GDT 中的 invalid entry，則下一次記憶體存取可能會導致 General Protection Fault。為了避免 General Protection Fault，將 Segment register 設為 `__BOOT_DS = 0x18`

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L112)
```x86asm=112
	movl	$__BOOT_DS, %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %fs
	movl	%eax, %gs
	movl	%eax, %ss
```

### Result: 完成後 CPU / kernel 處於什麼狀態？

Segment register `DS`、`ES`、`FS`、`GS`、`SS` 已設為 `__BOOT_DS = 0x18`，避免發生 General Protection Fault 的潛在可能性。

## Set the stack pointer 

### Purpose: 這段程式碼要完成什麼？
設定 stack pointer

### Problem: 為什麼需要這個操作？
後續指令需要使用 `lretl` 指令，重新載入 Code Segment。 

使用 `lretl` 指令，需要 stack 機制。所以需要設定 stack pointer。

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L120)
```x86asm=120
    leal	rva(boot_stack_end)(%ebp), %esp
```
把 `esp` 設為 `boot_stack_end` 的 記憶體位址 

## Reload the Code Segment from current GDT 

### Purpose: 這段程式碼要完成什麼？

重新載入 Code Segment

### Context: CPU / kernel 現在處於什麼狀態？

Code Segment register 並不是載入 protected mode 的 Code Segment Selector 

### Problem: 為什麼需要這個操作？

Code Segment Register 不能像其他 Segment Register 一樣。
直接使用 `movl` 修改 Code Segment Register 的值。
需要使用 `lretl` 機制修改 Code Segment Register。

### Mechanism: 這段程式碼使用什麼機制?

`lretl` 會 pop 出 stack 頂端的兩個值
第一個值由 `%eip` 接住
第二個值由 `%cs` 接住
透過這個機制修改 Code Segment Register


```
Stack
     +------------------+  
     |  Value for eip   | top
     |  Value for cs    |
     |                  |
     +------------------+  
```

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L122)
```x86asm=122
	pushl	$__KERNEL32_CS
	leal	rva(1f)(%ebp), %eax
	pushl	%eax
	lretl
1:
```

### Result: 完成後 CPU / kernel 處於什麼狀態？

Code Segment Register 已載入 `$__KERNEL32_CS` Code Segment Selector

## 參考來源

* [Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode 第 2 段 Reload the segments if needed](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4#reload-the-segments-if-needed)
* [Protected mode on x86-compatible processors](http://100.71.125.87:3000/NP0YD3GRTxSlQtAXr__Wrw)
