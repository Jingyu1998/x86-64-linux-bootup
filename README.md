# x86-64-linux-bootup
本筆記逐步展開開機流程。先將開機流程分成 firmware、bootloader、linux 三個階段。 再將 Linux 開機流程展開成 Kernel Setup、Kernel Initialization 與 Start init process 三個階段。  本筆記主要探討 Kernel Setup 階段，以 kernel source code 為主要依據，逐步追蹤各階段的程式碼與 CPU / kernel 狀態變化。  後續將繼續追蹤 Kernel Initialization，以及 Start init process。
