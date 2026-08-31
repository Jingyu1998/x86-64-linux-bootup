---
tags: early_boot, startup_32
---
# Stage 1: Clear CPU State and Compute runtime address of `startup_32` 

## Clear CPU State

### Purpose: 這段程式碼要完成什麼？
將 Direction Flag 和 Interrupt flag 清為 0

### Context: CPU / kernel 現在處於什麼狀態？

| Direction Flag | Interrupt flag |
| -------------- | -------------- |
| 不確定是否為0 | 不確定是否為0 |

### Problem: 為什麼需要這個操作？

- 確保後續程式碼需要用到 string operations 時，會對 index register, eg: `esi`、`edi` 採取 auto-increment 
- 不希望 CPU 在沒有有效的 interrupt table 或 interrupt handler 的情況下收到中斷

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L90)

```x86asm=90
	cld
	cli
```

### Result: 完成後 CPU / kernel 處於什麼狀態？

| Direction Flag | Interrupt flag |
| -------------- | -------------- |
| 0 | 0 |

## Compute runtime address of `startup_32`

### Purpose: 這段程式碼要完成什麼？
計算runtime時，`startup_32` symbol 的實際記憶體位址

### Context: CPU / kernel 現在處於什麼狀態？
kernel 不知道`startup_32` symbol 的實際記憶體位址

### Problem: 為什麼需要這個操作？

後續程式碼會使用 `startup_32` symbol，來計算其他 symbol 或 structure 相對於 `startup_32` symbol 的 offset 

### Mechanism: 這段程式碼使用什麼機制?

call 指令需要 stack 來儲存 return address
kernel 在此刻暫時使用 `boot_params` 裡的 4-byte `scratch` field 當作 stack

| Offset/Size | Proto | Name | Meaning |
| ----------- | ----- | ---- | --------|
| 1E4/004 | ALL | scratch | Scratch field for the kernel setup code |

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L93)

```x86asm=93
/*
 * Calculate the delta between where we were compiled to run
 * at and where we were actually loaded at.  This can only be done
 * with a short local call on x86.  Nothing  else will tell us what
 * address we are running at.  The reserved chunk of the real-mode
 * data at 0x1e4 (defined as a scratch field) are used as the stack
 * for this calculation. Only 4 bytes are needed.
 */
	leal	(BP_scratch+4)(%esi), %esp
	call	1f
1:	popl	%ebp
	subl	$ rva(1b), %ebp
```

* `leal (BP_scratch+4)(%esi), %esp`:<br>計算 `%esi + BP_scratch + 4` 的位址，並將該位址放入 `%esp`（Stack Pointer）。
    ```
    0x1E8        +------------------+  %esp
                 |  Scratch         |
    0x1E4        +------------------+ 
    ```


* `call 1f`:<br> 將 return address 寫入 `%esp` - 4，也就是 `BP_scratch` 的位置。
    ```
    0x1E8      +--------------------+  
               | runtime addr of 1: |
    0x1E4      +--------------------+  %esp
    ```
    
* `popl %ebp`:<br>pop the **return address, `1:`** to `%ebp`. 現在 `%ebp` 儲存 label `1:` 的 runtime address。

* `subl	$ rva(1b), %ebp`:<br>`%ebp` 減掉 label `1:` 相對於 symbol `startup_32` 的偏移量。現在 `%ebp` 儲存 symbol `startup_32` 的 runtime address。
	* `rva(1b)`: label `1:` 相對於 symbol `startup_32` 的偏移量。

### Result: 完成後 CPU / kernel 處於什麼狀態？
kernel 將 `startup_32` symbol 的實際記憶體位址儲存於 `ebp` 暫存器

## 參考來源
[Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode 第 2 段 Reload the segments if needed](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4#reload-the-segments-if-needed)