---
tags: early_boot, startup_64
---
# Stage 5: Load the Stage1 Interrupt Descriptor Table

## Boot Interrupt Descriptor Table Descriptor

[**SYM_DATA_START(boot_idt_desc) to SYM_DATA_END(boot_idt_desc)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L617)
```x86asm
SYM_DATA_START(boot_idt_desc)
	.word	boot_idt_end - boot_idt - 1
	.quad	0
SYM_DATA_END(boot_idt_desc)
```

`boot_idt_desc` 是提供 `lidt` 使用的 IDT Descriptor：

其中：
* `boot_idt_end - boot_idt - 1`: 計算 `boot_idt` 的大小減 1，作為 **IDT Limit**。
* `.quad 0`: 預留 IDT Base，之後 runtime 由 `load_stage1_idt()` 設定為 `boot_idt` 的位址。

## Boot Interrupt Descriptor Table

[**SYM_DATA_START(boot_idt) to SYM_DATA_END(boot_idt)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L622)
```x86asm
SYM_DATA_START(boot_idt)
	.rept	BOOT_IDT_ENTRIES
	.quad	0
	.quad	0
	.endr
SYM_DATA_END_LABEL(boot_idt, SYM_L_GLOBAL, boot_idt_end)
```

* `BOOT_IDT_ENTRIES` = 32
* Boot IDT 有 32 個 entry。
* 每個 IDT entry 由兩個 `.quad` 組成，共 16 bytes，並且初始化為 0。

## Load the Stage1 Interrupt Descriptor Table

### Purpose: 這段程式碼要完成什麼？

將 kernel 自己定義的 IDT 載入 IDTR。

### Context: CPU / kernel 現在處於什麼狀態？

進入 64-bit kernel Setup code 的方式
* 除了執行 32-bit kernel Setup code 的最後一條指令 `lret` ，接著跳轉到 64-bit kernel Setup code。
* 也可以是由 bootloader 自行將 CPU 切換到 64 bit long mode。並且將 CPU 控制權交給 kernel 從 64-bit kernel Setup code 開始執行。

若是採用第二種方式進入 long mode，那 kernel 仍使用在 bootloader 定義的 IDT。 

進入 64-bit kernel Setup code 後
- CPU 已經執行 `cli`，maskable interrupts 已關閉。
- kernel 尚未載入自己定義的 IDT。

### Problem: 為什麼需要這個操作？

後續 kernel setup code 需要使用 kernel 自己定義的 IDT，
而不是建立在 bootloader 所提供的 IDT environment 上。

因此需要將 IDTR 切換至 kernel 自己定義的 IDT。

### Implementation: 它實際怎麼完成？

[**SYM_CODE_START(startup_64) to SYM_CODE_END(startup_64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L377)
```x86asm=377
	/*
	 * RSI holds a pointer to a boot_params structure provided by the
	 * loader, and this needs to be preserved across C function calls. So
	 * move it into a callee saved register.
	 */
	movq	%rsi, %r15

	call	load_stage1_idt
```

* `movq %rsi, %r15` : 將 `%rsi` 的值存入 `%r15` 
    * `%rsi` 原先存放 boot_params。
    * `%rsi` 屬於 caller-saved register。呼叫 C function 後，callee 可以使用 `%rsi`，因此不能依賴 `%rsi` 的值保持不變。
    * 所以將 `%rsi` 的值存入 `%r15`。`%r15` 是屬於 callee-saved register。C function 若要使用 `%r15`，需要負責保存並恢復 `%r15`。
* `call	load_stage1_idt`: 呼叫 c function `load_stage1_idt`

[**function load_stage1_idt in `arch/x86/boot/compressed/idt_64.c`**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/idt_64.c)

```c=30
/* Setup IDT before kernel jumping to  .Lrelocated */
void load_stage1_idt(void)
{
	boot_idt_desc.address = (unsigned long)boot_idt;


	if (IS_ENABLED(CONFIG_AMD_MEM_ENCRYPT))
		set_idt_entry(X86_TRAP_VC, boot_stage1_vc);

	load_boot_idt(&boot_idt_desc);
}
```

* `boot_idt_desc.address = (unsigned long)boot_idt;`
    * boot IDT 的 runtime address 載入 IDT Descriptor 的 base 欄位 
* `load_boot_idt(&boot_idt_desc);` : 最終透過 `lidt` 指令載入 IDT Descriptor。
    * IDT Descriptor 載入 IDTR 。

在目前不考慮 `CONFIG_AMD_MEM_ENCRYPT=y` 的情況下，此時 IDT 的所有 entry 都是空的，並不包含任何 valid interrupt handler。

### Result: 完成後 CPU / kernel 處於什麼狀態？

IDTR 已指向 kernel 自己定義的 IDT。

## 參考來源

* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 1-5 段 Load of the Interrupt Descriptor Table](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#load-of-the-interrupt-descriptor-table)
* [Calling Convention](../prior-knowledge/64-bit/Calling-Convention.md)