---
tags: early_boot, .Lrelocated
---
# Stage 3: Build identity mapping page table

## Purpose: 這段程式碼要完成什麼？

建立新的 **identity mapping page table**，讓 kernel decompressor 後續執行時，需要使用的 memory region 都具有有效的 virtual-to-physical mapping。

## Context: CPU / kernel 現在處於什麼狀態？

* Kernel 已完成 relocation。
* CPU 已跳轉至 relocated kernel code。
* CPU 目前使用的 page table 可能是：
    * bootloader 建立的 boot page table
    * 32-bit setup code 建立的 boot page table。
* Kernel 已經重新載入 Stage2 IDT。
* 接下來即將建立新的 identity mapping page table。

## Problem: 為什麼需要這個操作？

Kernel relocation 後，原本使用的 boot page table 可能被 decompressor 覆寫。

因此，即使之前已經建立過 early page table，kernel 仍需要重新建立或擴充 page table，確保 decompressor 執行期間所需的 memory mapping 不會因為原本的 page table 被覆寫而失效。

新的 page table 使用 **identity mapping：** $Virtual Address$ = $Physical Address$

因此對於建立 identity mapping 的 memory region，virtual address 經過 page table translation 後會對應到相同的 physical address。

## Implementation: 它實際怎麼完成？

[SYM_FUNC_START_LOCAL_NOALIGN(.Lrelocated) to SYM_FUNC_END(.Lrelocated)](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L467)
```x86asm=467
	/* Pass boot_params to initialize_identity_maps() */
	movq	%r15, %rdi
	call	initialize_identity_maps
```

* `movq %r15, %rdi` : 將 `%r15` 的值存入 `%rdi` 
    * `%r15` 存放 boot_params。
    * 提供 `%rdi` 作為 `initialize_identity_maps` 的第一個引數
* `call	initialize_identity_maps`: 呼叫 c function `initialize_identity_maps`

大致閱讀 function `initialize_identity_maps` 的實作。並將函式實作分成以下主題來探討

* 初始化 mapping_info 
* 擴充或建立新的 identity mapping page table
* 建立 kernel decompressor 所需的 identity mappings
* 將新的 top-level page table 位址寫入 CR3

## 初始化 mapping_info 

變數 mapping_info 是結構體 x86_mapping_info 的 instance 。

### 結構體設計目的 

結構體 x86_mapping_info 提供給建立 memory mapping 的程式碼使用的**設定**與**資源管理資訊**。

### 結構體實作
[struct x86_mapping_info in `/arch/x86/include/asm/init.h`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/include/asm/init.h#L7)
```c
struct x86_mapping_info {
	void *(*alloc_pgt_page)(void *); /* allocate buf for page table */
	void (*free_pgt_page)(void *, void *); /* free buf for page table */
	void *context;			 /* context for alloc_pgt_page */
	unsigned long page_flag;	 /* page flag for PMD or PUD entry */
	unsigned long offset;		 /* ident mapping offset */
	bool direct_gbpages;		 /* PUD level 1GB page support */
	unsigned long kernpg_flag;	 /* kernel pagetable flag override */
};
```

* `alloc_pgt_page`: 當 mapping code 需要建立新的 page table page 時，要**呼叫哪個 function** 來取得 memory。
* `free_pgt_page`: 當 mapping code 需要釋放 page table page，要使用哪個 function。
* `context`: 提供 `alloc_pgt_page` 使用的 context。context 包含 page table allocation area 的位址、大小與目前使用的 offset。
* `page_flag`: 建立 PMD/PUD large-page mapping 時，這個 mapping 的 page entry 要使用這些 flags。
* `offset`: 表示 identity mapping 的 offset
* `direct_gbpages`: 是否可以使用 1GB large page 建立 direct mapping
* `kernpg_flag `:  kernel page table hierarchy 中，指向下一層 page table 的 entry 要使用什麼 flags。

### 初始化結構體

[function `initialize_identity_maps` in `/arch/x86/boot/compressed/ident_map_64.c`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/ident_map_64.c#L118)
```c=118
	/* Init mapping_info with run-time function/buffer pointers. */
	mapping_info.alloc_pgt_page = alloc_pgt_page;
	mapping_info.context = &pgt_data;
	mapping_info.page_flag = __PAGE_KERNEL_LARGE_EXEC | sme_me_mask;
	mapping_info.kernpg_flag = _KERNPG_TABLE;
```

## 擴充或建立新的 identity mapping page table

讀取目前 CR3 中的 top-level page table，並與 `_pgtable` 比較：

[function `initialize_identity_maps` in `/arch/x86/boot/compressed/ident_map_64.c`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/ident_map_64.c#L131)
```c=131
	/*
	 * If we came here via startup_32(), cr3 will be _pgtable already
	 * and we must append to the existing area instead of entirely
	 * overwriting it.
	 *
	 * With 5-level paging, we use '_pgtable' to allocate the p4d page table,
	 * the top-level page table is allocated separately.
	 *
	 * p4d_offset(top_level_pgt, 0) would cover both the 4- and 5-level
	 * cases. On 4-level paging it's equal to 'top_level_pgt'.
	 */
	top_level_pgt = read_cr3_pa();
	if (p4d_offset((pgd_t *)top_level_pgt, 0) == (p4d_t *)_pgtable) {
		pgt_data.pgt_buf = _pgtable + BOOT_INIT_PGT_SIZE;
		pgt_data.pgt_buf_size = BOOT_PGT_SIZE - BOOT_INIT_PGT_SIZE;
		memset(pgt_data.pgt_buf, 0, pgt_data.pgt_buf_size);
	} else {
		pgt_data.pgt_buf = _pgtable;
		pgt_data.pgt_buf_size = BOOT_PGT_SIZE;
		memset(pgt_data.pgt_buf, 0, pgt_data.pgt_buf_size);
		top_level_pgt = (unsigned long)alloc_pgt_page(&pgt_data);
	}
```

### if 條件成立

* **if 條件成立**代表目前使用的是 32-bit setup code `startup_32` 建立的 boot page table，kernel 會沿用 boot page table 並在 boot page table 結尾處向後建立新的 identity mapping page table。

```
_pgtable
    │
    ├───────────────┬───────────────────────────────┐
    │               │                               │
    │ BOOT_INIT_    │ 剩餘空間                       │
    │ PGT_SIZE      │                               │
    │               │                               │
    └───────────────┴───────────────────────────────┘
                    ↑
                    pgt_data.pgt_buf
```

`BOOT_INIT_PGT_SIZE` 代表 32-bit setup code `startup_32` **已使用的 page-table 區域大小**。

* `pgt_data.pgt_buf = _pgtable + BOOT_INIT_PGT_SIZE;` 
    * 計算建立新的 identity mapping page table 的起始位址
* `pgt_data.pgt_buf_size = BOOT_PGT_SIZE - BOOT_INIT_PGT_SIZE;`
    * 剩餘可以用來配置 identity mapping page table 的空間大小
* `memset(pgt_data.pgt_buf, 0, pgt_data.pgt_buf_size);`
    * 將後續用來建立 identity mapping page table 的記憶體空間先清為 0

### if 條件不成立

```
_pgtable
    │
    ├──────────────────────────────────────────────┐
    │                                              │
    │ BOOT_PGT_SIZE                                │
    │                                              │
    │                                              │
    └──────────────────────────────────────────────┘
    ↑
    pgt_data.pgt_buf
```

從 `_pgtable` 開始建立新的 identity mapping page table

* `pgt_data.pgt_buf = _pgtable;` 
    * identity mapping page table 的起始位址為 `_pgtable`
* `pgt_data.pgt_buf_size = BOOT_PGT_SIZE ;`
    * 用來配置 identity mapping page table 的空間大小
* `memset(pgt_data.pgt_buf, 0, pgt_data.pgt_buf_size);`
    * 將後續用來建立 identity mapping page table 的記憶體空間先清為 0
* `top_level_pgt = (unsigned long)alloc_pgt_page(&pgt_data);`
    * 配置新的 top-level page table，並將其位址存入 `top_level_pgt`。

## 建立 kernel decompressor 所需的 identity mappings

[function `initialize_identity_maps` in `/arch/x86/boot/compressed/ident_map_64.c`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/ident_map_64.c#L154)
```c=154
	/*
	 * New page-table is set up - map the kernel image, boot_params and the
	 * command line. The uncompressed kernel requires boot_params and the
	 * command line to be mapped in the identity mapping. Map them
	 * explicitly here in case the compressed kernel does not touch them,
	 * or does not touch all the pages covering them.
	 */
	kernel_add_identity_map((unsigned long)_head, (unsigned long)_end);
	boot_params_ptr = rmode;
	kernel_add_identity_map((unsigned long)boot_params_ptr,
				(unsigned long)(boot_params_ptr + 1));
	cmdline = get_cmd_line_ptr();
	kernel_add_identity_map(cmdline, cmdline + COMMAND_LINE_SIZE);
```

`kernel_add_identity_map()` 會遍歷 page table hierarchy：

* 如果對應的 page table entry 已存在，則使用既有 entry。
* 如果不存在，則配置新的 page table entry。
* 新的 entry 使用前面 `mapping_info` 所設定的 page flags。

建立的 identity mapping 包含：

* kernel image：`_head` → `_end`
* boot parameters provided by the bootloader
* kernel command line provided by the bootloader

## 將新的 top-level page table 位址寫入 CR3

[function `initialize_identity_maps` in `/arch/x86/boot/compressed/ident_map_64.c`](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/ident_map_64.c#L154)
```c=154
	/* Load the new page-table. */
	write_cr3(top_level_pgt);
```

## Result: 完成後 CPU / kernel 處於什麼狀態？

* Kernel 已建立或擴充新的 identity mapping page table。
* Identity mapping 已涵蓋：
    * compressed kernel image
    * boot parameters
    * kernel command line
* CR3 已更新為新的 top-level page table。
* CPU 後續執行 decompressor 時，可以透過新的 identity mapping page table 存取上述必要的 memory regions。

## 參考來源

* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 2 段 The last actions before the kernel decompression](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#the-last-actions-before-the-kernel-decompression)
