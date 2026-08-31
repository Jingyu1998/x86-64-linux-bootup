---
tags: early_boot, .Lrelocated
---
# Stage 2: Load the Stage2 Interrupt Descriptor Table

## Purpose: 這段程式碼要完成什麼？

在 Interrupt Descriptor Table 設定 PF 和 NMI Entries,
並重新載入 IDT 位址到 IDTR

## Context: CPU / kernel 現在處於什麼狀態？

* Kernel 已完成 relocation。
* CPU 已跳轉至 relocated kernel code。
* Kernel 已將 Interrupt Flag 清為 0。停用 maskable interrupt。

## Problem: 為什麼需要這個操作？

後續使用 `initialize_identity_maps` 函式，重新初始化 Boot Page Table 時，
可能會觸發 Page Fault, 因此需要準備 Page Fault Interrupt Handler。

Kernel 雖然已將 Interrupt Flag 清為 0，但是後續執行 decompression 時，CPU 仍可能收到 NMI 。NMI 不受 Interrupt Flag 控制，至少讓 CPU 有一個有效的 IDT entry 可以跳轉，而不是因為沒有有效 handler 而導致更嚴重的 fault。 

## Implementation: 它實際怎麼完成？

[SYM_FUNC_START_LOCAL_NOALIGN(.Lrelocated) to SYM_FUNC_END(.Lrelocated)](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L465)
```x86asm=465
call	load_stage2_idt
```

[`void load_stage2_idt(void)` in `/arch/x86/boot/compressed/idt_64.c` ](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/idt_64.c#L59)
```c=59
void load_stage2_idt(void)
{
	boot_idt_desc.address = (unsigned long)boot_idt;

	set_idt_entry(X86_TRAP_PF, boot_page_fault);
	set_idt_entry(X86_TRAP_NMI, boot_nmi_trap);

#ifdef CONFIG_AMD_MEM_ENCRYPT
...
#endif

	load_boot_idt(&boot_idt_desc);
}
```

* `boot_idt_desc.address = (unsigned long)boot_idt;`
    * boot IDT 的 runtime address 載入 IDT Descriptor 的 base 欄位 
* `set_idt_entry(X86_TRAP_PF, boot_page_fault);` : 設定 Page Fault entry
* `set_idt_entry(X86_TRAP_NMI, boot_nmi_trap);`: 設定 NMI entry
* `load_boot_idt(&boot_idt_desc);` : 最終透過 `lidt` 指令載入 IDT Descriptor。
    * IDT Descriptor 載入 IDTR 。

## Result: 完成後 CPU / kernel 處於什麼狀態？

* kernel 已載入新的 IDT, 此時
    * IDT 具有 Page Fault Interrupt Handler， 用來處理 CPU 發生 memory access 失敗。
    * IDT 具有 NMI entry, 至少讓 CPU 有一個有效的 IDT entry 可以跳轉。

## 參考來源

* [Page Fault Interrupt Handler](http://100.71.125.87:3000/rPnM4yf1QqC9LfEwpW0gew)
* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 2 段 The last actions before the kernel decompression](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#the-last-actions-before-the-kernel-decompression)