# x86-64-linux-bootup
本筆記逐步展開開機流程。先將開機流程分成 firmware、bootloader、linux 三個階段。 再將 Linux 開機流程展開成 Kernel Setup、Kernel Initialization 與 Start init process 三個階段。

本筆記主要探討 Kernel Setup 階段，以 kernel source code 為主要依據，逐步追蹤各階段的程式碼與 CPU / kernel 狀態變化。

後續將繼續追蹤 Kernel Initialization，以及 Start init process。

## Boot process
系統開機過程
- Firmware → Bootloader → Linux 

```mermaid
%%{init: {
  'flowchart': { 
    'nodeSpacing': 40, 
    'rankSpacing': 40,
    'curve': 'linear'
  },
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': 'monospace'
  }
}}%%
flowchart TD
    %% Define Styles
    classDef StringNode fill:#95CACA
    
    %% Subgraph
    subgraph Firmware
        direction TB
        string4[["Bios"]]:::StringNode
        string5[["UEFI"]]:::StringNode
    end

    %% Nodes
    string2[["Bootloader"]]:::StringNode
    string3[["Linux"]]:::StringNode

    %% Connection
    Firmware --> string2 --> string3
```

## Linux boot process
Linux 開機過程可以向下細分為 
- Kernel Setup → Kernel Initialization -> Start PID-1 process

完成開機後，Linux 進入可供使用者或應用程式使用的正常運行狀態。

### Kernel Setup

Kernel Setup 目的
- 準備 Kernel decompression 所需的執行環境
- 將 compressed kernel image 解壓縮至記憶體

Kernel Setup 完成事項
- 將 CPU 從 protected mode 切換到 long mode
- 將 compressed kernel image 搬移至 relocation target
- 解壓縮 compressed kernel image 

Kernel Setup 程式碼階段
- 32-bit Kernel Setup → 
- 64-bit Kernel Setup → 
- Relocated Kernel Setup → 

### Kernel Initialization

Kernel Initialization 目的
- 準備 userspace 程式可以在 kernel 正常運行所需的執行環境

Kernel Initialization 完成事項
- 初始化 core subsystem
- 設定 memory management
- 偵測 hardware
- 載入 driver

### Start init process

Start init process 目的
- 啟動 Linux userspace 的第一個 process, PID-1。
- 將後續 userspace process 所需的初始化工作交給 PID-1

Start init process 完成事項
- PID-1 執行系統的 init process，例如 `systemd`
- 由 PID-1 啟動其他必要的 userspace processes
    - 系統日誌, `systemd-journald`
    - 網路管理, `NetworkManager` 

```mermaid
%%{init: {
  'flowchart': { 
    'nodeSpacing': 40, 
    'rankSpacing': 15,
    'curve': 'linear'
  },
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': 'monospace'
  }
}}%%
flowchart TD
    %% Define Styles
    classDef StringNode fill:#95CACA

    %% Subgraph 
    subgraph Linux
        direction TB
        %% Subgraph 
        subgraph Kernel-Setup
            direction TB
            string1[["32-bit kernel setup"]]:::StringNode
            string2[["64-bit kernel setup"]]:::StringNode
            string3[["relocated kernel setup"]]:::StringNode

            %% Connection
            string1 --> string2 --> string3 
        end
        
        %% Subgraph 
        subgraph Kernel-Initialization
            direction TB
            string4[["many stage in kernel initialization"]]:::StringNode
        end
        
        %% Subgraph 
        subgraph Start-Init-Process
            direction TB
            string5[["PID 1 starts other userspace processes"]]:::StringNode
        end
        
        %% Connection 
        Kernel-Setup --> Kernel-Initialization 
        Kernel-Initialization --> Start-Init-Process
    end
```

## Scope

本筆記以 x86_64 Linux 搭配 GRUB2 作為 bootloader 的開機流程為主要範例。

GRUB2 負責載入 Compressed Kernel Image，並完成 32-bit Linux boot protocol 所要求的前置準備。GRUB2 完成準備後, 將 CPU 控制權交給 Kernel Setup Code。

本筆記的內容聚焦在**開機流程**的 **Kernel Setup 階段**。

完成 Kernel Setup 階段後，Kernel Setup Code 已將 Compressed Kernel Image 解壓縮到記憶體。並將 CPU 控制權交給 Decompressed Kernel。

## Reading list

本筆記的內容主要參考 Linux-insides。

GRUB2 完成的前置準備恰好對應 Linux-insides Booting Chapter 第 4 篇的起始狀態。

以下為閱讀清單:
* [Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4)
* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5)

## Prior Knowledge for 32-bit Kernel Setup
- [Real mode on x86-compatible processors](prior-knowledge/32-bit/Real-mode-on-x86-compatible-processors.md)
- [Protected mode on x86-compatible processors](prior-knowledge/32-bit/Protected-mode-on-x86-compatible-processors.md)
- [Linker Script](prior-knowledge/32-bit/Linker-Script.md)
- [x86 Paging](prior-knowledge/32-bit/x86-Paging.md)

## 32-bit Kernel Setup

內容來源:
[Linux-insides Booting Chapter 第 4 篇 Transition to 64-bit mode](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-4)

32-bit Kernel Setup 目的
- 準備進入 long mode 的執行環境
- 將 kernel 從 protected mode 切換到 long mode

32-bit Kernel Setup 完成事項
- 載入 kernel 自己定義的 GDT
- 確定 CPU 支援 long mode
- 計算出 compressed kernel image 的 relocation target。不過 64-bit Kernel Setup 會再重算一次。
- CPU 啟用 PAE mode in CR4
- 建立 boot page table，CR3 指向 boot page table 起始位址。
- CPU 啟用 Long Mode Enable in EFER
- CPU 啟用 Paging in CR0
- 跳轉至 64-bit kernel Setup 

32-bit Kernel Setup 程式碼階段
- [Stage 1: Clear CPU State and Compute runtime address of `startup_32`](32-bit/1-Clear-CPU-State-and-Compute-runtime-addr-of-startup-32.md)
- [Stage 2: Reload GDT and Reset Segment Register](32-bit/2-Reload-GDT-and-Reset-Segment-Register.md)
- [Stage 3: CPU verification](32-bit/3-CPU-verification.md)
- [Stage 4: Calculate the Relocation Target](32-bit/4-Calculate-Relocation-Target.md)
- [Stage 5: Enable PAE mode](32-bit/5-Enable-PAE-mode.md)
- [Stage 6: Set up paging](32-bit/6-Set-up-paging.md)
- [Stage 7: Transition to 64-bit](32-bit/7-Transition-to-64-bit.md)

32-bit Kernel Setup 程式碼範圍
* [**SYM_FUNC_START(startup_32) to SYM_FUNC_END(startup_32)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L82)

```mermaid
%%{init: {
  'flowchart': { 
    'nodeSpacing': 40, 
    'rankSpacing': 20,
    'curve': 'linear'
  },
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': 'monospace'
  }
}}%%
flowchart TD
    %% Define Styles
    classDef StringNode fill:#95CACA

    %% Subgraph 
    subgraph 32-bit kernel Setup
        direction TB
        string1[["Clear CPU State<br/>Compute runtime address of startup_32"]]:::StringNode
        string2[["Reload GDT and Reset Segment Register"]]:::StringNode
        string3[["CPU verification"]]:::StringNode
        string4[["Calculate the Relocation Target"]]:::StringNode
        string5[["Enable PAE mode"]]:::StringNode
        string6[["Set up paging"]]:::StringNode
        string7[["Transition to 64-bit"]]:::StringNode
        
        %% Connection
        string1 --> string2 --> string3 --> string4  
        string4 --> string5 --> string6 --> string7 
    end
```

## Prior Knowledge for 64-bit Kernel Setup
[Calling Convention](prior-knowledge/64-bit/Calling-Convention.md)

## 64-bit Kernel Setup
內容來源:
* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 1 段 First steps in the long mode](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#first-steps-in-the-long-mode)

64-bit Kernel Setup 目的
- 準備 kernel relocation 的執行環境
- 將 compressed kernel image 搬移至 relocation target

64-bit Kernel Setup 完成事項
- 計算出 compressed kernel image 的 relocation target。
- 載入 kernel 自己定義的 GDT
- 載入 kernel 自己定義的 IDT
- 將 compressed kernel image 搬移至 relocation target
- 跳轉至 relocated kernel Setup 

64-bit Kernel Setup 程式碼階段
- [Stage 1: Clear CPU State](64-bit/1-Clear-CPU-State.md)
- [Stage 2: Clear data segment registers](64-bit/2-Clear-data-segment-registers.md)
- [Stage 3: Calculate the Relocation Target](64-bit/3-Calculate-Relocation-Target.md)
- [Stage 4: Reload GDT and Reset Segment Register](64-bit/4-Reload-GDT-and-Reset-Code-Segment-Register.md)
- [Stage 5: Load the Stage1 Interrupt Descriptor Table](64-bit/5-Load-the-Stage1-Interrupt-Descriptor-Table.md)
- [Stage 6: Kernel relocation](64-bit/6-Kernel-relocation.md)
- [Stage 7: Reload GDT after kernel relocation](64-bit/7-Reload-GDT-after-kernel-relocation.md)
- [Stage 8: Jump on the relocated code](64-bit/8-Jump-on-the-relocated-code.md)	

64-bit Kernel Setup 程式碼範圍
* [**SYM_CODE_START(startup_64) to SYM_CODE_END(startup_64)**](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L286)

```mermaid
%%{init: {
  'flowchart': { 
    'nodeSpacing': 40, 
    'rankSpacing': 15,
    'curve': 'linear'
  },
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': 'monospace'
  }
}}%%
flowchart TD
    %% Define Styles
    classDef StringNode fill:#95CACA

    %% Subgraph 
    subgraph 64-bit kernel Setup
        direction TB
        string1[["Clear CPU State"]]:::StringNode
        string2[["Clear data segment registers"]]:::StringNode
        string3[["Calculate the Relocation Target"]]:::StringNode
        string4[["Reload GDT and Reset Segment Register"]]:::StringNode
        string5[["Load the Stage1 Interrupt Descriptor Table"]]:::StringNode
        string6[["Kernel relocation"]]:::StringNode
        string7[["Reload GDT after kernel relocation"]]:::StringNode
        string8[["Jump on the relocated code"]]:::StringNode
        
        %% Connection
        string1 --> string2 --> string3 --> string4  
        string4 --> string5 --> string6 --> string7
        string7 --> string8 
    end
```

## Prior Knowledge for Relocated Kernel Setup
* [BSS Section](prior-knowledge/Relocated/BSS-Section.md)
* [Page Fault Interrupt Handler](prior-knowledge/Relocated/Page-Fault-Interrupt-Handler.md)
* [Linux Kernel Page-table Abstraction](prior-knowledge/Relocated/Linux-Kernel-Page-table-Abstraction.md)
* [ELF, Executable and Linking Format](prior-knowledge/Relocated/ELF-Executable-and-Linking-Format.md)

## Relocated Kernel Setup
內容來源:
* [Linux-insides Booting Chapter 第 5 篇 Kernel decompression 第 2、3 段](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-5#the-last-actions-before-the-kernel-decompression)

Relocated Kernel Setup 目的
- 準備 kernel decompression 的環境
- 將 compressed kernel image 解壓縮至 memory

Relocated Kernel Setup 完成事項
- 初始化 BSS section
- 載入 kernel 自己定義的 IDT，此時 IDT 已設定 PF 和 NMI entry
- 建立或擴充 identity mapping page table
- 將 compressed kernel image 解壓縮，得到 ELF executable file `vmlinux`。
- 將 ELF 的 `PT_LOAD` segments **載入** physical memory。
- 修改已載入的 kernel image 裡**需要修正的 address**
- CPU 跳轉至 decompressed kernel entry point

Relocated Kernel Setup 程式碼階段
- [Stage 1: Initialize the BSS section](Relocated/1-Initialize-BSS-section.md)
- [Stage 2: Load the Stage2 Interrupt Descriptor Table](Relocated/2-Load-Stage2-Interrupt-Descriptor-Table.md)
- [Stage 3: Build identity mapping page table](Relocated/3-Build-identity-mapping-page-table.md)
- [Stage 4: Kernel decompression](Relocated/4-Kernel-decompression.md)
- [Stage 5: Jump on decompressed kernel entrypoint](Relocated/5-Jump-on-decompressed-kernel-entrypoint.md)

Relocated Kernel Setup 程式碼範圍
* [SYM_FUNC_START_LOCAL_NOALIGN(.Lrelocated) to SYM_FUNC_END(.Lrelocated)](https://elixir.bootlin.com/linux/v6.14/source/arch/x86/boot/compressed/head_64.S#L453)

```mermaid
%%{init: {
  'flowchart': { 
    'nodeSpacing': 40, 
    'rankSpacing': 30,
    'curve': 'linear'
  },
  'themeVariables': {
    'fontSize': '14px',
    'fontFamily': 'monospace'
  }
}}%%
flowchart TD
    %% Define Styles
    classDef StringNode fill:#95CACA

    %% Subgraph 
    subgraph Relocated Kernel Setup
        direction TB
        string1[["Initialize the BSS section"]]:::StringNode
        string2[["Load the Stage2 Interrupt Descriptor Table"]]:::StringNode
        string3[["Build identity mapping page table"]]:::StringNode
        string4[["Kernel decompression"]]:::StringNode
        string5[["Jump on decompressed kernel entrypoint"]]:::StringNode

        %% Connection
        string1 --> string2 --> string3 --> string4  
        string4 --> string5 
    end
```
