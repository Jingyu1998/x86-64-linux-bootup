---
tags: early_boot, startup_32
---
# Stage 4: Calculate the Relocation Target

## Compressed Kernel Image 的組成結構

The compressed kernel image mainly consists of two parts:

* Kernel's setup and decompressor code
* Chunk of compressed kernel code

## linker script 的前三個 section

Linker Script 是用來**描述** Linker 應該如何**安排 Compressed Kernel Image 的 memory layout** 的**腳本**。

[ linker script arch/x86/boot/compressed/vmlinux.lds.S ](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/vmlinux.lds.S)

```
SECTIONS
{
	/* Be careful parts of head_64.S assume startup_32 is at
	 * address 0.
	 */
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
	.rodata..compressed : {
		*(.rodata..compressed)
	}
	.text :	{
		_text = .; 	/* Text */
		*(.text)
		*(.text.*)
		*(.noinstr.text)
		_etext = . ;
	}
```

* `. = 0`：設定 linker 的 **location counter**，表示接下來從 `0` 開始配置。
* `.head.text`
    * `_head = .`: 建立代表 .head.text 起始位置的 linker symbol。
    * `HEAD_TEXT` 展開為 `KEEP(*(.head.text))`, 將各個 input object file 中的 `.head.text` section 放入這個 output section。  
        * `KEEP()`: 告訴 linker：即使這個 section 看起來沒有被其他程式碼引用，也不要把它丟掉。
    * `_ehead = .`：建立代表 .head.text 結束位置的 linker symbol。
* `.rodata..compressed`
    * `*(.rodata..compressed)`：將各個 input object file 中的 `.rodata..compressed` section 放入這個 output section。  
* `.text`
    * `_text = .`: 建立代表 .text 起始位置的 linker symbol。
    * `*(.text)` : 將各個 input object file 中的 `.text` section 放入這個 output section。
    * `*(.text.*)` : 將各個 input object file 中的 `.text.*` section 放入這個 output section
    * `*(.noinstr.text)` : 將各個 input object file 中的 `.noinstr.text` section 放入這個 output section。
    * `_etext = .`：建立代表 .text 結束位置的 linker symbol。

----

* `.head.text` - section with the `startup_32` 、`startup_64` and other setup code 
* `.rodata..compressed` - section with the compressed kernel code
* `.text` - section with the decompressor code


```
Compressed Kernel Image

├──────────────────────────┐   address = 0
│ .head.text               │
│                          │
│ startup_32               │
│ startup_64               │
│ other setup code         │
├──────────────────────────┤
│ .rodata..compressed      │
│                          │
│ compressed kernel code   │
├──────────────────────────┤
│ .text                    │
│                          │
│ decompressor code        │
└──────────────────────────┘
```

## decompression 機制介紹

kernel decompression 是在原地進行的，也就是在 compressed kernel 所在的同一位置進行。這意味著在 decompression 過程中，decompressed kernel image 會覆蓋compressed kernel image。

### Problem

在 decompression 過程中，如果 decompressed kernel image 覆蓋 decompressor code 或尚未解壓縮的 compressed kernel image，這將導致 decompressor code 或 compressed kernel image 損壞。

### Mechanism

避免此問題的方法是為 decompressed kernel image 分配一個 buffer，但將compressed kernel image 搬移到此 buffer 的末尾，並在此 buffer 的開頭留出一些空間來存放 decompressed kernel image 的各個部分。

kernel decompressor 必須選擇正確的**參數**，

這個正確的**參數**就是搬遷 compressed kernel image 的目標位址，將它稱為 relocation target。

在 decompression 過程中, 指向 decompressed kernel 末端的指標的移動速度才不會比目前指向 compressed kernel 的指標的移動速度快。

示意圖如下
![](https://0xax.gitbook.io/linux-insides/~gitbook/image?url=https%3A%2F%2F3490860827-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-x-prod.appspot.com%2Fo%2Fspaces%252F-M6zWfOAfArTn8XhUn_P%252Fuploads%252Fgit-blob-fbaf3456e64f43de7c3b2321a102afff1b2dee62%252Fkernel-relocation.svg%3Falt%3Dmedia&width=400&dpr=3&quality=100&sign=da55659d&sv=2)

decompressed kernel 的 buffer 起始位址由 `LOAD_PHYSICAL_ADDR` 巨集指定。
預設起始位址為 `0x01000000` 。

## Calculate the start address of buffer

### Purpose: 這段程式碼要完成什麼？

這段程式碼是要計算出放置 decompressed kernel image 的起始位址。

### Context: CPU / kernel 現在處於什麼狀態？

尚未計算出放置 decompressed kernel image 的起始位址

### Problem: 為什麼需要這個操作？

執行 decompression 前，需要準備 buffer。
此 buffer 用來存放 decompressed kernel image 。

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L138)

```x86asm=138
/*
 * Compute the delta between where we were compiled to run at
 * and where the code will actually run at.
 *
 * %ebp contains the address we are loaded at by the boot loader and %ebx
 * contains the address where we should move the kernel image temporarily
 * for safe in-place decompression.
 */

#ifdef CONFIG_RELOCATABLE
	movl	%ebp, %ebx
	movl	BP_kernel_alignment(%esi), %eax
	decl	%eax
	addl	%eax, %ebx
	notl	%eax
	andl	%eax, %ebx
	cmpl	$LOAD_PHYSICAL_ADDR, %ebx
	jae	1f
#endif
	movl	$LOAD_PHYSICAL_ADDR, %ebx
1:

	/* Target address to relocate to for decompression */
	addl    BP_init_size(%esi), %ebx
	subl	$ rva(_end), %ebx
```

* `%ebp` register 儲存 runtime startup_32 起始位址，也就是載入 protected kernel mode 的起始位址。
* `%ebp` 的值為`0x00100000`。 
* boot_params 提供對齊邊界 `BP_kernel_alignment` 。值為 `0x00200000`
* 將 `%ebx` 對齊至 `0x00200000` 的實作流程
	* `decl %eax` : Subtract 1 in `%eax`. In binary, this **turns the alignment bit off** and **turns all bits below it on**.<br>**Now `%eax` is `0x001FFFFF`**
    * `addl %eax, %ebx` : we are now pushed `%ebx` into the range of the next 2MB block.<br>**Now `%ebx` is `0x002FFFFF`**
	* `notl %eax` : Create the "Clear" Mask, `%eax` bit 0-20 is 0, bit 21-31 is 1.<br>**Now `%eax` is `0xFFE00000`**
	* `andl %eax, %ebx` : bitwise AND between our **pushed address** and our **clear mask**.<br>**Now `%ebx` is `0x00200000`**
* 對齊完成後，比較 `%ebx` 和 `$LOAD_PHYSICAL_ADDR`=`0x01000000` 
    * 如果 `%ebx` 等於或大於 LOAD_PHYSICAL_ADDRESS，則保持不變。
    * 否則，將 LOAD_PHYSICAL_ADDRESS 載入 `%ebx`

### Result: 完成後 CPU / kernel 處於什麼狀態？

已計算出放置 decompressed kernel image 的起始位址，起始位址存於 `%ebx`

## Calculate the relocation target of compressed kernel image

### Purpose: 這段程式碼要完成什麼？

這段程式碼計算 relocation target, 也就是搬遷 compressed kernel image 的目標位址。
 
### Context: CPU / kernel 現在處於什麼狀態？

已計算出放置 decompressed kernel image 的起始位址，起始位址存於 `%ebx`

### Problem: 為什麼需要這個操作？

計算出搬遷 compressed kernel image 的目標位址。確保後續 decompression 過程中，decompressed kernel image 不會覆蓋到尚未解壓縮的 compressed kernel image。

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L160)

```x86asm=160
	/* Target address to relocate to for decompression */
	addl    BP_init_size(%esi), %ebx
	subl	$ rva(_end), %ebx
```

* `%ebx` register 的值加上 `BP_init_size`=`0x040A6000` 
    * `BP_init_size` 來自 boot_params，代表存放 decompressed kernel image 的 buffer 大小。
    * 現在 `%ebx` 指向 buffer 尾端。
* `%ebx` register 的值減掉 compressed kernel image 的大小 
    * `rva(_end)` 代表 Symbol `_end` 相對於 `startup_32` 的 offset，也就是 compressed kernel image 的大小

**Memory layout:**
```plaintext
Buffer size = 0x040A6000

0x01000000 +----------------------------------+ <--- LOAD_PHYSICAL_ADDR
           |                                  | 
           |          Empty Workspace         |  
           |   (This is the "Runway" where    |
           |    decompressed data will grow)  |
           |                                  |
0x???????? +----------------------------------+ <--- %ebx (Relocation Target)
           |                                  |      startup_32
           |       Compressed Kernel Image    |
           |          (vmlinuz)               |      Size:
           |   (Moved here from its original  |      $ rva(_end)
           |       load address, 0x00100000 ) |      
           |                                  |
0x050A6000 +----------------------------------+ <--- Total Workspace End
```

### Result: 完成後 CPU / kernel 處於什麼狀態？

已計算出搬遷 compressed kernel image 的目標位址。

## 參考來源

* [Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode 第 3-2 段 Calculation of the kernel relocation address](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4#calculation-of-the-kernel-relocation-address)
* [Keyword Keep in Linker-script](https://wiki.osdev.org/Linker_Scripts#KEEP)
* [Linker Script](http://100.71.125.87:3000/sSqTuTc9QVGpb8B1vJ2uhA)