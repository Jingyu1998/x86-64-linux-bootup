---
tags: early_boot, startup_64
---
# Stage 3: Calculate the Relocation Target

## Calculate the start address of buffer

### Purpose: 這段程式碼要完成什麼？

這段程式碼是要計算出放置 decompressed kernel image 的起始位址。

### Context: CPU / kernel 現在處於什麼狀態？

進入 64-bit kernel Setup code 的方式
* 除了執行 32-bit kernel Setup code 的最後一條指令 `lret` ，接著跳轉到 64-bit kernel Setup code。
* 也可以是由 bootloader 自行將 CPU 切換到 64 bit long mode。並且將 CPU 控制權交給 kernel 從 64-bit kernel Setup code 開始執行。

若是採取第二種方式進入 64-bit kernel Setup code，
kernel 尚未計算出放置 decompressed kernel image 的起始位址。

### Problem: 為什麼需要這個操作？

執行 decompression 前，需要準備 buffer。
此 buffer 用來存放 decompressed kernel image 。

### Implementation: 它實際怎麼完成？

[**SYM_CODE_START(startup_64) to SYM_CODE_END(startup_64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L309)

```x86asm=309
	/*
	 * Compute the decompressed kernel start address.  It is where
	 * we were loaded at aligned to a 2M boundary. %rbp contains the
	 * decompressed kernel start address.
	 *
	 * If it is a relocatable kernel then decompress and run the kernel
	 * from load address aligned to 2MB addr, otherwise decompress and
	 * run the kernel from LOAD_PHYSICAL_ADDR
	 *
	 * We cannot rely on the calculation done in 32-bit mode, since we
	 * may have been invoked via the 64-bit entry point.
	 */

	/* Start with the delta to where the kernel will run at. */
#ifdef CONFIG_RELOCATABLE
	leaq	startup_32(%rip) /* - $startup_32 */, %rbp
	movl	BP_kernel_alignment(%rsi), %eax
	decl	%eax
	addq	%rax, %rbp
	notq	%rax
	andq	%rax, %rbp
	cmpq	$LOAD_PHYSICAL_ADDR, %rbp
	jae	1f
#endif
	movq	$LOAD_PHYSICAL_ADDR, %rbp
```

* `leaq	startup_32(%rip) /* - $startup_32 */, %rbp`:
    * 不同於 protected mode, long mode 使用 RIP addressing 的方式找出 symbol `startup_32` 的 runtime 位址。 
    * symbol  `startup_32` 的起始位址也就是 compressed kernel image 的起始位址。
    * 將此位址載入 `%rbp`
* 將 `%rbp` 對齊至 `BP_kernel_alignment` 
    * 細節參考: [32-bit Kernel Setup Stage 4: Calculate the Relocation Target](../32-bit/4-Calculate-Relocation-Target.md) 
* 對齊完成後，比較 `%rbp` 和 `$LOAD_PHYSICAL_ADDR`=`0x01000000` 
    * 如果 `%rbp` 等於或大於 LOAD_PHYSICAL_ADDRESS，則保持不變。
    * 否則，將 LOAD_PHYSICAL_ADDRESS 載入 `%rbp`

### Result: 完成後 CPU / kernel 處於什麼狀態？

不論 kernel 是由 32-bit setup code 或是 bootloader 切換至 long mode。 
kernel 都計算出放置 decompressed kernel image 的起始位址，起始位址存於 `%rbp`

## Calculate the relocation target of compressed kernel image

### Purpose: 這段程式碼要完成什麼？

這段程式碼計算 relocation target, 也就是搬遷 compressed kernel image 的目標位址。
 
### Context: CPU / kernel 現在處於什麼狀態？

已計算出放置 decompressed kernel image 的起始位址，起始位址存於 `%rbp`

### Problem: 為什麼需要這個操作？

計算出搬遷 compressed kernel image 的目標位址。確保後續 decompression 過程中，decompressed kernel image 不會覆蓋到尚未解壓縮的 compressed kernel image。

### Implementation: 它實際怎麼完成？

[**SYM_CODE_START(startup_64) to SYM_CODE_END(startup_64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L337)

```x86asm=337
	/* Target address to relocate to for decompression */
	movl	BP_init_size(%rsi), %ebx
	subl	$ rva(_end), %ebx
	addq	%rbp, %rbx
```

* 載入 `BP_init_size`=`0x040A6000` 到 `%rbx`
    * `BP_init_size` 來自 boot_params，代表存放 decompressed kernel image 的 buffer 大小。
* buffer 大小減掉 compressed kernel image 的大小 
    * `rva(_end)` 代表 Symbol `_end` 相對於 `startup_32` 的 offset，也就是 compressed kernel image 的大小
    * 此時 `%rbx` 作為 `rbp` 計算 Runtime Relocation Target 的 偏移量。 
* `%rbx` 加上  decompressed kernel image 的起始位址
    * 此時 `%rbx` 為 Relocation Target 的位址

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

kernel 計算出搬遷 compressed kernel image 的目標位址, 存於 `%rbx`。

## 參考來源

* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 1-3 段 Calculation of the kernel relocation address](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#calculation-of-the-kernel-relocation-address)
* [32-bit Kernel Setup Stage 4: Calculate the Relocation Target](../32-bit/4-Calculate-Relocation-Target.md)