---
tags: early_boot, Concept
---

# Linux Kernel Page-table Abstraction

## Purpose

Linux kernel 使用一套 generic page-table abstraction，
將不同 architecture 的 page-table hierarchy 統一成相同的介面。

Linux kernel 使用以下名稱表示 page-table hierarchy：

```text
PGD
 ↓
P4D
 ↓
PUD
 ↓
PMD
 ↓
PTE
```

這些名稱是 **Linux kernel 的 abstraction**，
並不直接代表特定 CPU architecture 的硬體 page-table level。

## Problem

不同 CPU architecture 的 page-table hierarchy 可能不同。

例如 x86_64 在 four-level paging 下，硬體 page-table hierarchy 為：

```
PML4
 ↓
PDPT
 ↓
PD
 ↓
PT
```

但 Linux kernel 的 generic page-table abstraction 固定提供：

```
PGD
 ↓
P4D
 ↓
PUD
 ↓
PMD
 ↓
PTE
```

如果 kernel source code 直接使用各 architecture 的硬體名稱，
memory management code 就必須針對不同 architecture 撰寫不同的 page-table 操作。

因此 Linux kernel 使用 generic page-table abstraction，
將 architecture-specific 的 page-table implementation 隔離起來。

## Linux Kernel Page-table Levels

Linux kernel 定義以下 page-table levels：

| Level | Linux kernel name | Purpose                  |
| ----: | ----------------- | ------------------------ |
|     5 | `PGD`             | Page Global Directory    |
|     4 | `P4D`             | Page 4th-level Directory |
|     3 | `PUD`             | Page Upper Directory     |
|     2 | `PMD`             | Page Middle Directory    |
|     1 | `PTE`             | Page Table Entry         |

