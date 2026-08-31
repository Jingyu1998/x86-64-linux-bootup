---
tags: early_boot, startup_32
---
# Stage 6: Set up paging

現在我們將看到 kernel 如何建立 early page table 以切換到 long mode。但在我們直接進入程式碼之前，我們需要記住一件重要的事情。

Compressed kernel image 將被 relocate 到儲存在 `ebx` 暫存器中的位址。因此，包括 **page table** 在內的所有結構都應該與 relocation target 對齊。

The page table structure for boot is defined in the same source code file and looks like this:
[**`/arch/x86/boot/compressed/head_64.S`**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L639)
```x86asm=639
/*
 * Space for page tables (not in .bss so not zeroed)
 */
	.section ".pgtable","aw",@nobits
	.balign 4096
SYM_DATA_LOCAL(pgtable,		.fill BOOT_PGT_SIZE, 1, 0)
```

## 將 page table 佔用的 memory area 填入 0

### Purpose: 這段程式碼要完成什麼？

將 page table 佔用的 memory area 填入 0。

### Context: CPU / kernel 現在處於什麼狀態？

已計算出 compressed kernel image 的 relocation target，並將其儲存在 `%ebx` 暫存器中。

### Problem: 為什麼需要這個操作？

提供填入 **page table entries** 所需的乾淨環境。

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L200)
```x86asm=200
	/* Initialize Page tables to 0 */
	leal	rva(pgtable)(%ebx), %edi
	xorl	%eax, %eax
	movl	$(BOOT_INIT_PGT_SIZE/4), %ecx
	rep	stosl
```

* `leal rva(pgtable)(%ebx), %edi`: 計算 `pgtable` label 的 physical address。 `pgtable` 位址儲存在 `%edi` 中，`%edi` 是 string operations 使用的 index register
* `xorl %eax, %eax`: set the `%eax` register to 0
* `movl $(BOOT_INIT_PGT_SIZE/4), %ecx`:
	* `%ecx` acts as the loop counter.
	* `BOOT_INIT_PGT_SIZE` 是保留給 boot page tables 的記憶體空間，大小為 24KB。
		* [`# define BOOT_INIT_PGT_SIZE	(6*4096)`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/include/asm/boot.h#L48)
    * 因為後續的 `stosl` 指令一次將 `%eax` 的 4-byte 值寫入 memory。所以迭代次數為 boot page tables 的記憶體空間大小除以 4，並存入 `%ecx`。
* `rep stosl`:
    *  重複執行 `stosl` 指令，直到 `%ecx` 變為 0。
    *  執行一次 `stosl` 指令，將 `%eax` 的值寫入到 `%edi` 所指向的記憶體位址。
    *  因為先前 **clear the CPU state** 階段，已透過 `cld` 將 Direction Flag 清為 0，所以每次執行後 `%edi` 自動增加 4。
    *  每次執行後 `%ecx` 自動減少 1。

### Mechanism: Repeat Store String Long 

使用 x86 string operation `rep stosl`，
重複將 `%eax` 的 32-bit value 寫入 `%edi` 指向的 memory，
並依據 Direction Flag 自動調整 `%edi`,
直到 `%ecx` 次數執行完畢。

由於 `%eax` 已設為 `0`，因此可以將整個 page table memory area 清為 0。

### Result: 完成後 CPU / kernel 處於什麼狀態？

完成後，`BOOT_INIT_PGT_SIZE` 所涵蓋的 page table memory area 已全部填入 0。
可以開始填入 **page table entries** 到 boot page tables。

## 填入 level4 page table 的 first entry

The **boot page table** will have the following structure:
* 1 level4 page table
* 1 level3 page table
* 4 level2 page table that maps everything with 2M pages

### Purpose: 這段程式碼要完成什麼？

填入 level4 page table 的 first entry。

### Context: CPU / kernel 現在處於什麼狀態？

已將 page table 佔用的 memory area 填入 0，準備開始建立 boot page table。

### Problem: 為什麼需要這個操作？

kernel 在切換至 long mode 前，必須準備好 boot page table。

### Implementation: 它實際怎麼完成？

Kernel 將 level4 page table 的 first entry 填入 level3 page table 的位址和 flag 。
level3 page table 的位址位於 `pgtable + 0x1000` ，flag 為 `0x007`。

In our case, the flags `0x7` are:
* Bit 0 Present: 1 = Valid
* Bit 1 Read/Write: 1 = Writable
* Bit 2 User/Supervisor: 1 = User access allowed

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L206)
```x86asm=206
	/* Build Level 4 */
	leal	rva(pgtable + 0)(%ebx), %edi
	leal	0x1007 (%edi), %eax 
	movl	%eax, 0(%edi)
	addl	%edx, 4(%edi)
```

* `leal rva(pgtable + 0)(%ebx), %edi` : `%edi` 指向 level4 page table 的 base address
* `leal 0x1007(%edi), %eax`: 計算 `pgtable + 0x1007`，並將結果存入 `%eax`。
    * `0x1000` 是 Level3 page table 相對於 Level4 page table 的 offset。
    * `0x007` 是 page table entry 的 flags。
    * 因此 `%eax` 的值為 `Level3 page table address | 0x007`。
* `movl %eax, 0(%edi)`: 將 `%eax` 寫入 `%edi` 指向的記憶體位址。此時完成寫入**level4 page table 的 first entry** 的 lower 32 bit。
* `addl	%edx, 4(%edi)`: 將 `%edx` 加入 `%edi + 4` 指向的記憶體位址。此時完成寫入**level4 page table 的 first entry** 的 higher 32 bit。
    * 此時 `%edx` = 0, 參考來源 [ `xorl	%edx, %edx` ](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L181)

### Result: 完成後 CPU / kernel 處於什麼狀態？

完成寫入 first entry 到 level4 的 boot page table。該 entry 指向位於 `pgtable + 0x1000` 的 Level 3 page table，並包含 flag `0x007`。

## 填入 level3 page table 的 first 4 entry

### Purpose: 這段程式碼要完成什麼？

kernel 填入 level3 page table 的 first 4 entry。

### Context: CPU / kernel 現在處於什麼狀態？

完成寫入 first entry 到 level4 的 boot page table。

### Problem: 為什麼需要這個操作？

kernel 在切換至 long mode 前，必須準備好 boot page table。

### Implementation: 它實際怎麼完成？

kernel 將 level3 page table 的 first 4 entry 填入 level2 page table 的 base address 以及 flag `0x007`。

level3 page table 的 first entry 位於 level4 page table 起始位置偏移 0x1000 處。

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L212)
```x86asm=212
	/* Build Level 3 */
	leal	rva(pgtable + 0x1000)(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx 
1:	movl	%eax, 0x00(%edi)
	addl	%edx, 0x04(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi 
	decl	%ecx
	jnz	1b
```

* `leal rva(pgtable + 0x1000)(%ebx), %edi` : `%edi` 指向 level3 page table 的 base address, 也就是 first entry 的位址。
* `leal 0x1007(%edi), %eax`: 計算 `%edi + 0x1007`，並將結果存入 `%eax`。
    * `0x1000` 是第一個 Level2 page table 相對於 Level3 page table 的 offset。
    * `0x007` 是 page table entry 的 flags。
    * 因此 `%eax` 的值為 `Level2 page table address | 0x007`。
* `movl $4, %ecx`: This sets the loop counter to 4.
* The Loop (`1:`):
    * `movl %eax, 0(%edi)`: 將 `%eax` 寫入 `%edi` 指向的記憶體位址。此時完成寫入**level3 page table entry** 的 lower 32 bit。
    * `addl	%edx, 4(%edi)`: 將 `%edx` 加入 `%edi + 4` 指向的記憶體位址。此時完成寫入**level3 page table entry** 的 higher 32 bit。
        * 此時 `%edx` = 0, 參考來源 [ `xorl	%edx, %edx` ](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L181)
    * `addl $0x00001000, %eax`: 將 `%eax` 增加 `0x1000`，使其指向下一個 Level2 page table，同時保留 `0x007` flags。
    * `addl $8, %edi`: `%edi` 增加 8，使其指向 Level3 page table 的下一個 entry。
    * `decl %ecx`: Decrements the counter `%ecx`
    * `jnz 1b`: loop until `%ecx` = 0

### Result: 完成後 CPU / kernel 處於什麼狀態？

完成寫入 first 4 entry 到 level3 的 boot page table。四個 entry 個別指向第 1-4 個Level2 page table 的位址，並包含 flag `0x007`。

## 填入 4 個 level2 page table 的 entry

### Purpose: 這段程式碼要完成什麼？

kernel 填入 4 個 level2 page table 的 entry。
一個 page table 填入 512 個 entry，所以總共填入 2048 個 entry

### Context: CPU / kernel 現在處於什麼狀態？

完成寫入 first entry 到 level4 的 boot page table。
完成寫入 first 4 entry 到 level3 的 boot page table。

### Problem: 為什麼需要這個操作？

kernel 在切換至 long mode 前，必須準備好 boot page table。

### Implementation: 它實際怎麼完成？

4 個 Level 2 page table 位於 `pgtable + 0x2000` 開始的 memory area。每一個 Level 2 page table 包含 512 個 entry，每個 entry 映射 2MB 的 physical memory，並帶有以下 flag。

* Bit 0 Present: 1 = Valid
* Bit 1 Read/Write: 1 = Writable
* Bit 7 Page Size: 1 = 2MB Large Page, 省略 Level1 page table。
* Bit 8 G: 1 = 告訴 CPU 在執行 `movl` to `CR3` 指令時，不要使該 page 對應的 TLB entry 失效。

因此，Level 2 page table entry 的初始值為：
`0x00000183 = physical address 0x00000000 | flags 0x183`

在位址轉換過程中，**page-walk procedure** 在 **level2 page table 停止**，virtual address 的**低 21 bit**作為 2M-bytes page 內的**偏移量**。

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L223)
```x86asm=223
	/* Build Level 2 */
	leal	rva(pgtable + 0x2000)(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:	movl	%eax, 0(%edi)
	addl	%edx, 4(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

* `leal rva(pgtable + 0x2000)(%ebx), %edi`: `%edi` 指向 level2 page table 的 base address, 也就是 first entry 的位址。
* `movl $0x00000183, %eax`: 建立 level2 page table 的 first entry，physical address 為 0，並包含 `0x183` flags。
* `movl $2048, %ecx`: The counter is set to 2048.
* The Loop (`1:`):
    * `movl %eax, 0(%edi)`: 將 `%eax` 寫入 `%edi` 指向的記憶體位址。此時完成寫入**level2 page table entry** 的 lower 32 bit。
    * `addl	%edx, 4(%edi)`: 將 `%edx` 加入 `%edi + 4` 指向的記憶體位址。此時完成寫入**level2 page table entry** 的 higher 32 bit。
    * `addl $0x00200000, %eax`: 將 `%eax` 增加 `0x200000`，使其指向下一個 2M-Bytes physical address，同時保留 `0x183` flags。
    * `addl $8, %edi`: `%edi` 增加 8，使其指向 Level2 page table 的下一個 entry。
	* `decl %ecx`: Decrements the counter `%ecx`
	* `jnz 1b`: loop until `%ecx` = 0


### Result: 完成後 CPU / kernel 處於什麼狀態？

建立完成 boot page table。

4 個 Level 2 page table 共包含 2048 個 entries，每個 entry 映射一個 2MB physical memory page，因此總共建立：

`2048 × 2MB = 4GB`

的 identity mapping。

## Boot Page Table Memory layout

```plaintext=
PHYSICAL RAM ADDRESS
       (Relative to pgtable base)

       +0x0000 +----------------------------------+
               |  Level 4:                        |
               |  (1 active entry)                |
               +----------------------------------+
               | [0...7]: Pointer to Level 3      |==+ 
               |                                  |  |
               | [8...4095]: (All Zeros)          |  |
               +----------------------------------+  |
                                                     |
       +0x1000 +----------------------------------+  |
               |  Level 3:                        |<=+
               |  (4 active entries)              |
               +----------------------------------+
               | [0...7]:  Pointer to Level2 PT 0 |
               |                                  |====+
               | [8...15]: Pointer to Level2 PT 1 |    |
               |                                  |====|==+
               | [16...23]: Pointer to Level2 PT 2|    |  |
               |                                  |====|==|==+
               | [24...31]: Pointer to Level2 PT 3|    |  |  |
               |                                  |====|==|==|==+
               | [32...4095]: (All Zeros)         |    |  |  |  |
               +----------------------------------+    |  |  |  |
                                                       |  |  |  |
       +0x2000 +----------------------------------+    |  |  |  |
               | Level 2: PT 0 (512 entries)      |<===+  |  |  |
               | (Maps Physical 0MB to 1024MB)    |       |  |  |
               +----------------------------------+       |  |  |
               | [0...7]: Maps Phys 0x00000000    |       |  |  |
               |                                  |       |  |  |
               | [8...15]: Maps Phys 0x00200000   |       |  |  |
               |                                  |       |  |  |
               | ...                              |       |  |  |
               |[4088..4095]:Maps Phys 0x3FE00000 |       |  |  |
               |                                  |       |  |  |
               +----------------------------------+       |  |  |
                                                          |  |  |
       +0x3000 +----------------------------------+       |  |  |
               | Level 2: PT 1 (512 entries)      |<======+  |  |
               | (Maps Physical 1024MB to 2048MB) |          |  |
               +----------------------------------+          |  |
               | [0...7]: Maps Phys 0x40000000    |          |  |
               | ...                              |          |  |
               +----------------------------------+          |  |
                                                             |  |
       +0x4000 +----------------------------------+          |  |
               | Level 2: PT 2 (512 entries)      |<=========+  |
               | (Maps Physical 2048MB to 3072MB) |             |
               +----------------------------------+             |
               | [0...7]: Maps Phys 0x80000000    |             |
               | ...                              |             |
               +----------------------------------+             |
                                                                |
       +0x5000 +----------------------------------+             |
               | Level 2: PT 3 (512 entries)      |<============+
               | (Maps Physical 3072MB to 4096MB) |
               +----------------------------------+
               | [0...7]: Maps Phys 0xC0000000    |
               | ...                              |
               +----------------------------------+

       (Total Reserved Paging Space: 24KB)
```

## 將 boot page table 的 base address 載入 CR3

### Purpose: 這段程式碼要完成什麼？

將 boot page table 的 base address 填入 `cr3` register

### Context: CPU / kernel 現在處於什麼狀態？

建立完成 boot page table。
Boot Page table 一共映射 4GB 的 physical memory area 

### Problem: 為什麼需要這個操作？

CPU 必須知道 boot page table 的 base address，才能在後續啟用 paging 後，從該 page table hierarchy 開始進行 virtual address 到 physical address 的轉換。

### Implementation: 它實際怎麼完成？

[**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L234)
```x86asm=234
	/* Enable the boot page tables */
	leal	rva(pgtable)(%ebx), %eax
	movl	%eax, %cr3
```

### Result: 完成後 CPU / kernel 處於什麼狀態？

`CR3` 已指向 boot page table 的 Level4 page table。
下一步將進行切換至 long mode 所需的 CPU 設定。

## 參考資料
- [Paging](http://100.71.125.87:3000/f1Bjy9lkSB2KRSUQqLapSg)
- [Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode 第 4 段 Set up paging](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4#set-up-paging)

