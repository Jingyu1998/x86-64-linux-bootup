---
tags: early_boot, startup_64
---
# Stage 7: Reload GDT after kernel relocation

## 64-bit GDT descriptor

[**SYM_DATA_START_LOCAL(gdt64) to SYM_DATA_END(gdt64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L600)

```x86asm
	.data
SYM_DATA_START_LOCAL(gdt64)
	.word	gdt_end - gdt - 1   /*  GDTR Limit */
	.quad   gdt - gdt64         /*  64-bit GDTR Base */  
SYM_DATA_END(gdt64)
```

## Reload GDT after kernel relocation
### Purpose: 這段程式碼要完成什麼？

將 GDT 位址重新載入 GDTR

### Context: CPU / kernel 現在處於什麼狀態？

已將 compressed kernel image 複製到 relocation target

### Problem: 為什麼需要這個操作？

Kernel relocation 過程中，原先儲存 GDT 的記憶體可能在 kernel relocation 或後續 decompression 過程中被覆寫。

因此需要將 Relocation 後的 GDT 位址重新載入 GDTR

### Implementation: 它實際怎麼完成？

[**SYM_CODE_START(startup_64) to SYM_CODE_END(startup_64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L435)
```x86asm=435
	/*
	 * The GDT may get overwritten either during the copy we just did or
	 * during extract_kernel below. To avoid any issues, repoint the GDTR
	 * to the new copy of the GDT.
	 */
	leaq	rva(gdt64)(%rbx), %rax
	leaq	rva(gdt)(%rbx), %rdx
	movq	%rdx, 2(%rax)
	lgdt	(%rax)
```

`%rbx` 是 compressed kernel image 的 relocation target

![](http://100.71.125.87:3000/uploads/upload_ce0793617882af6acd9694b48667043d.png)

* `leaq	rva(gdt64)(%rbx), %rax`: 取得 kernel relocation 後 Symbol `gdt64` 的 runtime address。
* `leaq	rva(gdt)(%rbx), %rdx`: 取得 kernel relocation 後 Symbol `gdt` 的 runtime address。
* `movq	%rdx, 2(%rax)`: 
    * 這個指令將 **`%rdx` 的值**載入 **`%rax + 2` 指向的記憶體位址**。
    * 此時 GDT Descriptor 的 base address 欄位被設為 Symbol `gdt` 的 runtime address。 

* `lgdt (%rax)`: 這個指令載入 GDT Descriptor，更新 GDTR，使 GDTR 指向新的 GDT。


### Result: 完成後 CPU / kernel 處於什麼狀態？

kernel relocation 後，GDTR 已更新為指向 relocation 後的 GDT runtime address。

## 參考來源

[Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 1-6 段 Kernel relocation](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#kernel-relocation)