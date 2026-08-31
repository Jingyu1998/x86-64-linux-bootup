---
tags: early_boot, Concept
---

# Calling Convention

Calling conventions 是 application binary interface,ABI 的一部分。

Calling conventions 是指函式之間進行呼叫時，這些函式應該遵守的規範。

例如，它們之間如何**傳遞參數**與**回傳值**，以及如何處理 registers。

## x86-64 Calling Convention

x86-64 有 16 個 general purpose registers，如下表。所謂的 **general purpose registers**，是指在**撰寫 assembly 時**，作為**一般用途使用的 registers**。

然而，**C 語言**將 **general purpose registers** 賦予**特定的用途**。

當一個 C 函式 `caller` 要呼叫另一個 C 的函式名叫 `callee` 時，`caller` 會**將參數存放**在 `%rdi`、`%rsi` 等 registers，而 `callee` 就可以透過這些 registers 取得參數。

所以，`caller` 和 `callee` 必須遵守**哪些 registers 被用來傳遞參數**，以及**這些 registers 是傳遞第幾個參數**等規範。這些**規範就是 calling conventions。**

以System V AMD64 ABI 為例
* `Callee` 的前 6 個參數是 `caller` 透過 `%rdi`、`%rsi`、`%rdx`、`%rcx`、`%r8`、以及 `%r9` 依序傳遞給 `callee`。
* `Callee` 的回傳值是透過 `%rax` 回傳給 `caller`。

## Callee-Saved and Caller-Saved Registers

Callee-saved registers: `Callee` 負責儲存和恢復這些 registers 的值。

* `callee` 要使用這些 registers 的話，`callee` 負責將 register 的值保存在 stack 中。
* 當 `callee` 要結束回到 `caller` 之前， `callee` 必須從 stack 中恢復這些 registers 的值。

Caller-saved registers: `Caller` 負責儲存和恢復這些 registers 的值。
* 當 `caller` 呼叫 `callee` 之後，`callee` 可以任意使用這些 register。
* 如果 `caller` 在呼叫 `callee` 後仍需要這些 registers 原本的值，`caller` 必須在呼叫 `callee` 前保存這些 registers 的值。
* 當 `callee` 結束回到 `caller` 後，`caller` 可以從 stack 中回復這些 registers 的值。

| 64-bit | 32-bit | Special Purpose	| Caller-saved | Callee-saved | 
| ------ | ------ | --------------- | :----------: | :----------: | 
| rax | eax  | Return value  | $O$ |     |
| rbx | ebx  |               |     | $O$ |
| rcx | ecx  | 4th argument  | $O$ |     |
| rdx | edx  | 3rd argument  | $O$ |     |
| rsi | esi  | 2nd argument  | $O$ |     |
| rdi | edi  | 1st argument  | $O$ |     |
| rbp | ebp  | Frame pointer |     | $O$ |
| rsp | esp  | Stack pointer |     | $O$ |
| r8  | r8d  | 5th argument  | $O$ |     |
| r9  | r9d  | 6th argument  | $O$ |     |
| r10 | r10d |               | $O$ |     |
| r11 | r11d |               | $O$ |     |
| r12 | r12d |               |     | $O$ |
| r13 | r13d |               |     | $O$ |
| r14 | r14d |               |     | $O$ |
| r15 | r15d |               |     | $O$ |