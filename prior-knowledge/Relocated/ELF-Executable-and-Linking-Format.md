---
tags: early_boot, Concept
---

# ELF, Executable and Linking Format

ELF 是 Unix-like systems 廣泛使用的一種 **binary file format**。

ELF 可以描述不同類型的 binary object，例如：
* executable object file
* relocatable object file (`.o`)
* shared object file (`.so`)
* core dump

ELF 定義了一套標準的結構，讓 linker 和 loader 能夠理解 **binary file** 中：
* machine code
* data
* program segments
* symbol information
* relocation information

等資訊。

因此，ELF 不只是「程式碼本身」，而是**描述如何使用這些程式碼與資料的檔案格式**。

## Linking View 與 Execution View

ELF 為了同時支援 linking 與 execution，我們可以從兩種不同的觀點來理解**同一個 ELF file**：

* Linking View：以 **Section** 為主要結構。
* Execution View：以 **Segment** 為主要結構。

在 Execution 觀點下, 一個 Segment 通常是數個 Section 的組合，這些 Sections 具有**相容的載入需求與屬性**。

下圖顯示了這兩種不同觀點的結構圖
![](http://ccckmit.wdfiles.com/local--files/lk:elf/ELF2view.jpg)

如上圖所示: ELF 檔案採用兩個不同用途的 **header table**。
* 在 Linking 觀點下,  ELF 採用 **Section Header Table**, 這個 header table 記載了 section 資訊。
* 在 Execution 觀點下, ELF 採用 **Program Header Table**, 這個 header table 記載了 segment 資訊, 因此也可稱為 Segment Header Table。

### ELF Header 

ELF Header 儲存
* Program Header Table 的起始位址
* Section Header Table 的起始位址

ELF Header 的實作如下

[ struct Elf32_Ehdr in elf/elf.h in glibc ](https://elixir.bootlin.com/glibc/glibc-2.44/source/elf/elf.h#L63)

```c
typedef struct
{
  unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info, ELF 辨識代號區 */
  Elf32_Half	e_type;			/* Object file type, 檔案類型代號 */
  Elf32_Half	e_machine;		/* Architecture, 機器平台代號 */
  Elf32_Word	e_version;		/* Object file version, 版本資訊 */
  Elf32_Addr	e_entry;		/* Entry point virtual address, 程式的起始位址 */
  Elf32_Off	e_phoff;		/* Program header table file offset */
  Elf32_Off	e_shoff;		/* Section header table file offset */
  Elf32_Word	e_flags;		/* Processor-specific flags */
  Elf32_Half	e_ehsize;		/* ELF header size in bytes, ELF header 大小 */
  Elf32_Half	e_phentsize;		/* Program header table entry size */
  Elf32_Half	e_phnum;		/* Program header table entry count */
  Elf32_Half	e_shentsize;		/* Section header table entry size */
  Elf32_Half	e_shnum;		/* Section header table entry count */
  Elf32_Half	e_shstrndx;		/* Section header string table index */
} Elf32_Ehdr;
```

*  `e_phoff` 欄位指向 Program Header Table
*  `e_shoff` 欄位指向 Section Header Table

透過這兩個欄位，我們可以取得兩種 Header Table 資訊。 

## Linking View

**Linking View** 是從 **linker** 的角度來看 ELF。

在 linking 時期，ELF 主要以 **Section** 組織程式碼、資料、符號與 relocation 等資訊。

例如：
```
ELF File 由底下的 section 組成
│
├── .text       → machine code section
├── .rodata     → read-only data section
├── .data       → initialized data section
├── .bss        → uninitialized static variables
├── .symtab     → symbol table
├── .strtab     → string table
├── .rela.*     → relocation information
└── ...
```

Linker 可以根據 ELF 提供的這些 **section** 的資訊，進行：
* section 的組合
* symbol resolution
* relocation

Linker 最終產生 executable 或 shared object。

> **Linking View 關注的是「ELF 中有哪些 section，以及 linker 如何處理這些 section」。**

### Section Header Table

ELF Header 中的 `e_shoff` 欄位指向 Section Header Table。
在 Linking 觀點下,  ELF 主要透過 Section Header Table 描述 sections。

每一個 section 都有對應的 Section Header Table，描述該 section 的資訊。
資訊內容包含
* section type
* section flags
* section virtual address at execution
* section file offset


下圖顯示了如何透過 Section Header Table 讀取 Section  的方法。
![](http://ccckmit.wdfiles.com/local--files/lk:elf/ELFsectionheader.jpg)

Section Header Table 裡的 entry 實作如下

[ struct Elf32_Shdr in elf/elf.h in glibc ](https://elixir.bootlin.com/glibc/glibc-2.44/source/elf/elf.h#L383)

```c
typedef struct
{
  Elf32_Word	sh_name;		/* Section name (string tbl index) */
  Elf32_Word	sh_type;		/* Section type */
  Elf32_Word	sh_flags;		/* Section flags */
  Elf32_Addr	sh_addr;		/* Section virtual addr at execution */
  Elf32_Off	sh_offset;		/* Section file offset 在目的檔中的位址 */
  Elf32_Word	sh_size;		/* Section size in bytes */
  Elf32_Word	sh_link;		/* Link to another section */
  Elf32_Word	sh_info;		/* Additional section information */
  Elf32_Word	sh_addralign;		/* Section alignment */
  Elf32_Word	sh_entsize;		/* Entry size if section holds table */
} Elf32_Shdr;

```

## Linker 的工作

可以把 linker 的工作簡化理解成兩個階段：
* Symbol Resolution
* Relocation

**Symbol Resolution**
先找出：
```
symbol name
      ↓
symbol definition
      ↓
symbol 的最終 value / address
```

例如
```
global_variable
        ↓
global.o
        ↓
0x1000
```

**Relocation**
接著找到需要修改的位置：
```
.rela.text
    │
    ├── r_offset → 0x200
    ├── symbol   → global_variable
    └── type     → 某種 relocation type
```

Linker 根據 symbol 的最終位置與 relocation type，計算應該寫入的值，最後修改對應位置。

> Symbol information 告訴 linker「這個 symbol 是誰、在哪裡」；
> Relocation information 告訴 linker「哪裡需要根據這個 symbol 進行修正」。

### Symbol Section

Linker 不只需要知道有哪些 `.text`、`.data` 等 sections，也需要知道程式碼與資料中使用的 **symbols**。ELF 使用 `.symtab` section 儲存一般的 symbol table。

為什麼需要 symbol table 由以下情境說明:

:::info
```c
extern int global_variable;

int foo(void)
{
    return global_variable;
}
```

Compiler 在產生 object file 時，可能還不知道 `global_variable` 最終會被放在什麼位址。
:::

因此 ELF 需要透過 **Symbol Table** 描述：

* symbol 的名稱
* symbol 的位址
* symbol 的大小
* symbol 的類型
* symbol 的 binding
* symbol 所屬的 section

Symbol table 裡的 **entry** 都由一個 `Elf32_Sym` 結構描述。

**Symbol table entry** 的實作如下

[ struct Elf32_Sym in elf/elf.h in glibc ](https://elixir.bootlin.com/glibc/glibc-2.44/source/elf/elf.h#L520)

```c
typedef struct
{
  Elf32_Word	st_name;		/* Symbol name (string tbl index) */
  Elf32_Addr	st_value;		/* Symbol value */
  Elf32_Word	st_size;		/* Symbol size */
  unsigned char	st_info;		/* Symbol type and binding */
  unsigned char	st_other;		/* Symbol visibility */
  Elf32_Section	st_shndx;		/* Section index */
} Elf32_Sym;
```

主要欄位可以理解為：
* `st_name`：符號名稱在 string table 中的 index
* `st_value`: 符號的值，在已完成 linking 的 executable 中通常可以代表符號的位址。
* `st_size`: 符號的大小，以 byte為單位
* `st_info`: 細分為 bind 與 type 兩欄位
* `st_shndx`: 符號所在的 Section index

### Relocation Section

為什麼需要 Relocation ? 由以下情境說明:

:::info
Compiler 在編譯單一 `.c` source file 時，通常無法知道某些 symbol 最終的位址。

```c
extern int global_variable;

int foo(void)
{
    return global_variable;
}
```

Compiler 產生 machine code 時，需要在某個位置放入 `global_variable` 的位址。
但此時 `global_variable` 的最終位址尚未確定

因為它可能來自另一個 object file
```
foo.o
   │
   └── reference → global_variable

global.o
   │
   └── definition → global_variable
```

**Linker** 將兩個 object files 組合時，才能決定`global_variable` 的最終位址
:::

因此 **object file 必須記錄**：
> **哪一個位置需要**在 linking 時**被修正**，以及要**根據哪一個 symbol 進行修正**。

ELF 在 relocation sections 儲存 relocation table。

常見 section 形式包括：
* `.rel.text`
* `.rel.data`

Relocation table 裡的 entry 使用 `Elf32_Rel` 結構, 實作如下

[ struct Elf32_Rel in elf/elf.h in glibc ](https://elixir.bootlin.com/glibc/glibc-2.44/source/elf/elf.h#L635)

```c
typedef struct
{
  Elf32_Addr	r_offset;		/* Address */
  Elf32_Word	r_info;			/* Relocation type and symbol index */
} Elf32_Rel;
```

* `r_offset` 指出哪一個位置需要進行 relocation。也就是 **linker** 找到 relocation record 後，可以知道應該修改哪個位置。
* `r_info` 同時包含 symbol index、relocation type。因此 linker 可以知道 `r_offset` 標示的位置要根據哪一個 symbol，以及使用哪一種 relocation 方式進行修正。

## Execution View

**Execution View** 則是從 **loader** 的角度來看 ELF。
執行程式時，loader 並不需要知道 linker 使用的所有細節，例如：

```
.symtab
.strtab
.rela.text
.debug
```

loader 真正關心的是：**哪些內容需要被載入記憶體，以及載入後應該具有什麼屬性。**


具有相容載入需求的 sections 會被組合成 segment
* `.text`, `.rodata`, `.hash`, `.dynsym`, `.dynstr`, `.plt`, `.rel.got` 等 Section 會被併入到 Text Segment 當中
* `.data`, `.dynamic`, `.got`, `.bss` 等 Section 則會被併入到 Data Segement 當中。

```
ELF File
│
├── TEXT Segment 
│   ├── .text
│   ├── .rodata
│   └── ...
│
├── DATA Segment 
│   ├── .data
│   ├── .bss
│   └── ...
│
└── ...
```

> **Loader 關注的是「哪些資料需要被載入，以及它們在記憶體中如何配置」。**

### Program Header Table

ELF Header 中的欄位 `e_phoff` 指向 Program Header Table。
在 Execution 觀點下, 主要透過 Program Header Table 描述 ELF 中的 segments。

每一個 segment 都有對應的 Program Header，描述該 segment 的資訊。
資訊內容包含
* Segment type
* Segment file offset
* Segment virtual address
* Segment physical address

下圖顯示了如何透過 Program Header Table 讀取 Segment 的方法。
![](http://ccckmit.wdfiles.com/local--files/lk:elf/ELFprogramHeader.jpg)

Program Header Table 裡的 entry 實作如下

[ struct Elf32_Phdr in elf/elf.h in glibc ](https://elixir.bootlin.com/glibc/glibc-2.44/source/elf/elf.h#L383)

```c
typedef struct
{
  Elf32_Word	p_type;			/* Segment type */
  Elf32_Off	p_offset;		/* Segment file offset */
  Elf32_Addr	p_vaddr;		/* Segment virtual address */
  Elf32_Addr	p_paddr;		/* Segment physical address */
  Elf32_Word	p_filesz;		/* Segment size in file */
  Elf32_Word	p_memsz;		/* Segment size in memory */
  Elf32_Word	p_flags;		/* Segment flags */
  Elf32_Word	p_align;		/* Segment alignment */
} Elf32_Phdr;
```

* `p_type`：指定這個 segment 的類型。
* `p_offset`: segment 在 ELF file 中的起始位置。
* `p_vaddr`：segment 載入 memory 後的 virtual address。
* `p_filesz`：segment 在 file 中的大小。
* `p_memsz`：segment 載入 memory 後所需要的大小。
* `p_flags`：segment 的 memory permissions。
* `p_align`：segment 在 file 與 memory 中的 alignment。

## Loader 的工作

Loader 的工作

* 將 ELF 中需要執行的內容載入到記憶體，建立對應的 process memory layout，並將控制權交給程式。

### 常見的 Segment 

執行 ELF loading 時，Loader 使用的是以 Segment 為主的 Execution View。

常見的 Segment type 包括：

| Segment type | 對應 Section                    | 說明                      |
| ------------ | ------------------------------ | ------------------------ |
| `PT_PHDR`    | Program Header Table           | 描述 Program Header Table |
| `PT_INTERP`  | `.interp`                          | 指定 program interpreter  |
| `PT_LOAD`    | `.text`、`.rodata`、`.data`、`.bss` 等 | 需要載入記憶體的內容      |
| `PT_DYNAMIC` | `.dynamic`                         | 提供 dynamic linker 所需的資訊 |

### PT_LOAD segment

通常會有多個 `PT_LOAD` segments，讓不同屬性的區域可以分開，例如：

```
PT_LOAD
    │
    ├── .text
    ├── .rodata
    └── ...
    
PT_LOAD
    │
    ├── .data
    ├── .dynamic
    ├── .got
    └── .bss
```

這樣可以讓 code、read-only data 與 writable data 等不同區域具有不同的 memory permissions。

### Linux ELF loader

在 Linux 中，ELF loading 主要由 Kernel 負責。 Kernel 會解析 ELF 的 Program Header Table，並根據其中的 `PT_LOAD`，建立 process 的 memory layout。

可以將 ELF loading 過程簡化為：
1. Loader 根據 Program Header Table 找出所有 `PT_LOAD`。
2. 將 `PT_LOAD` segments **映射**到 Process Virtual Address Space
    * 讓程式的 code、data 等內容出現在 ELF 所指定的 virtual address。
    * 同時設定相應的 memory permissions，例如 `R-X`、`RW-`。
3. 跳轉到 ELF Header 的 e_entry
    * ELF Header 的 `e_entry` 指定程式的 entry point。
    * Loader 完成必要的載入與記憶體配置後，將控制權交給 `e_entry`。

## 參考來源

[目的檔格式 (ELF) - 陳鍾誠的網站](http://ccckmit.wikidot.com/lk:elf)

