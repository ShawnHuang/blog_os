#一個最小的 x86 kernel

[translated from](http://os.phil-opp.com/multiboot-kernel.html)

這篇文章中會解釋如何建立一個最小的 x86 作業系統的 kernel。事實上，這個 kernel 只會 boot 並且在螢幕上印出 `OK`。之後的 blog 系列文章中會使用 [Rust](http://www.rust-lang.org/) 程式語言來擴充這個 kernel。

我會盡量讓說明越詳細越好，並且讓程式碼盡可能的簡潔。如果你有任何的問題、建議或是 issues，你可以留言或是在 Github 上建立 [issue](https://github.com/phil-opp/blog_os/issues)。這些原始碼也都公開放在 [Github](https://github.com/phil-opp/blog_os/tree/multiboot_bootstrap/src/arch/x86_64) 上。

請注意以下教學只針對 Linux 所寫。針對 OS X 系統上仍然有些問題，詳細請看下面留言或是 Github 上的這個 [issue](https://github.com/phil-opp/blog_os/issues/55)。
如果你想要使用虛擬機來開發的話，你可以參考 Ashley Willams 的 [x86-kernel 專案](https://github.com/ashleygwilliams/x86-kernel)，裡面有近一步的說明。


###Overview

當你開機時，電腦會從一些特別的 flash memory 載入 [BIOS 韌體](https://en.wikipedia.org/wiki/BIOS)。BIOS 會自己針對硬體進測試及初始化程序，接下尋找可以 boot 的設備。如果有找到設備，控制權就會轉移到設備上的 bootloader，這是一小段可執行的程式儲存在設備的一開始位置。Bootloader 須找到設備上 kernel image 的位置，並且將他載入到記憶體。之後 bootloader 再將 CPU 轉為 [protected mode](https://en.wikipedia.org/wiki/Protected_mode)，這是因為 x86 CPU 一開始預設是在 [real mode](http://wiki.osdev.org/Real_Mode) 啟動 (為了相容一些 1978 時候的程式)。

這裡不會去寫一個 bootloader 出來，因為那是一個滿複雜的專案 (如果你真的想去做，可以參考 [Rolling Your Own Bootloader](http://wiki.osdev.org/Rolling_Your_Own_Bootloader))。我們會直接去使用[一些已經寫好的 bootloader](https://en.wikipedia.org/wiki/Comparison_of_boot_loaders)，但在各種不同的 bootloader 中要挑哪個來用呢？

###Multiboot

很幸運的，bootloader 有個設計標準：[Multiboot 規格書](https://en.wikipedia.org/wiki/Multiboot_Specification)。我們的 kernel 只需要標示成支援 Multiboot 即可，如此一來所有依 Multiboot 規定實作的 bootloader 都可以 boot 這個 kernel。接下來我們將參考 Multiboot 2 規格書 ([PDF](http://nongnu.askapache.com/grub/phcoder/multiboot.pdf))，並且使用 [GRUB 2](http://wiki.osdev.org/GRUB_2) 來作為 bootloader。

為了標示成可支援 bootloader，我們的 kernel 開頭必須要有 Multiboot Header，格式按照下面描述：

| Field         | Type            | Value                                     |
| ------------- |:---------------:| -----------------------------------------:|
| magic number  | u32             | `0xE85250D6`                              |
| architecture  | u32             | `0` for i386, `4` for MIPS                |
| header length | u32             | total header size, including tags         |
| checksum      | u32             | `-(magic + architecture + header_length)` |
| tags          | variable        |                                           |
| end tag       | (u16, u16, u32) | (0, 0, 8)                                 |

轉成 x86 組合語言之後，看起來會像如此 (Intel 語法)：

```asm
section .multiboot_header
header_start:
    dd 0xe85250d6                ; magic number (multiboot 2)
    dd 0                         ; architecture 0 (protected mode i386)
    dd header_end - header_start ; header length
    ; checksum
    dd 0x100000000 - (0xe85250d6 + 0 + (header_end - header_start))

    ; insert optional multiboot tags here

    ; required end tag
    dw 0    ; type
    dw 0    ; flags
    dd 8    ; size
header_end:
```

如果你不懂 x86 組合語言，這裡做個簡單說明：

* header 必須寫成 section 的形式，並且命名成 `.multiboot_header` (我們之後會用到)
* `header_start` 跟 `header_end` 是一種 label，標記著記憶體的位置，我們用這兩個 label 來計算 header 的長度 
* `dd` 代表 `define double` (32bit) ，而 `dw` 代表 `define word` (16bit)。這些指令只是去配置 32bit/16bit 大小的常數
* 在 checksum 計算中使用了一個額外的值 `0x100000000`，這是一個小技巧 [1] 來避免編譯警告

這時候我們可以用 `nasm` 組譯這個 asm 檔 (命名為 `multiboot_header.asm`)，他預設是產生了一個 flat binary (不具有 header 的二進位檔)，這個檔大小為 24 bytes (在 x86 機器上是以 little endian 表示)。

```
> nasm multiboot_header.asm
> hexdump -x multiboot_header
0000000    50d6    e852    0000    0000    0018    0000    af12    17ad
0000010    0000    0000    0008    0000
0000018
```

### The Boot Code

為了可以 boot 我們的 kernel，我們還需要加上一些程式碼，讓 bootloader 可以呼叫。因此我們建立一個 `boot.asm`：

```asm
global start

section .text
bits 32
start:
    ; print `OK` to screen
    mov dword [0xb8000], 0x2f4b2f4f
    hlt
```

這裡有一些新的指令需介紹：

* `global` 是用來 export 一個 label (讓這個 label 被公開)。而這個名為 `start` 的 label 將會是我們的 kernel 的進入點，所以他必須要被公開。
* `.text` section 是一個預設的 section，用來存放可執行的程式碼。
* `bits 32` 定義接下來的程式碼都是 32-bit 的指令。這是必要的定義，因為當 GRUB 啟動我們所寫的 kernel 時， CPU 就已經在 [Protected mode](https://en.wikipedia.org/wiki/Protected_mode) 下。下一篇教學中，我們會將 CPU 再轉換成 [Long mode](https://en.wikipedia.org/wiki/Long_mode)，此時我們可以使用 `bits 64` 來定義 (64-bit 指令)。
* `mov dword` 這個指令是將 32bit 的常數 `0x2f4f2f4b` 搬到位址為 `b8000` 的記憶體空間 (最後這道指令會印出 `OK` 在螢幕上，在下一篇教學會有更細部說明)。
* `hlt` 是一道停止的指令，會讓 CPU 停止執行。

透過組譯、反組譯之後，對照二進位以及反組譯結果，我們可以清楚看到 CPU [Opcodes](https://en.wikipedia.org/wiki/Opcode) 是如何實踐的：

```
> nasm boot.asm
> hexdump -x boot
0000000    05c7    8000    000b    2f4b    2f4f    00f4
000000b
> ndisasm -b 32 boot
00000000  C70500800B004B2F  mov dword [dword 0xb8000],0x2f4b2f4f
         -4F2F
0000000A  F4                hlt
```

### Building the Executable

為了要讓 GRUB 可以 boot 我們所寫的 kernel，kernel 就必須要是一個 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) executable。所以我們要用 `nasm` 先編譯成 ELF	[object 檔](http://wiki.osdev.org/Object_Files)，而不是編譯成單純的二進位檔，如何做呢？我們只需要代入 `-f elf64` 的引數到 `nasm` 就好了。

```
* ELF(Executable and Linkable Format) 是一種檔案格式，用於 excecutable 的執行檔、relocatable 的 object 檔以及動態連結函式庫等等
* 經過 linker (ld) 所產生的檔案為 executable，所以要編譯成 ELF executable，必須要先編譯成 ELF object 檔再用 ld 產生成 ELF executable。
```

在要建立 ELF executable，我們必須先將所有的 object 檔 [link](https://en.wikipedia.org/wiki/Linker_(computing)) 起來，這裡我們使用自訂義的 [linker 腳本](https://sourceware.org/binutils/docs/ld/Scripts.html) `linker.ld`：

```
ENTRY(start)

SECTIONS {
    . = 1M;

    .boot :
    {
        /* ensure that the multiboot header is at the beginning */
        *(.multiboot_header)
    }

    .text :
    {
        *(.text)
    }
}
```

我們仔細來看這段腳本：

* `start` 是一個程式進入點，當 kernel 載入到記憶體時，bootloader 就會跳到這裡執行
* `. = 1M;` 這一行將要載入 kernel 的起始位址設定在 1 MiB 處，而會這樣設定也只是個慣例。
* 這個 executable 有兩個 sections：前面是 `.boot` 而後面是 `.text`。
* `.text` 會產生一個 `.text` section 包含所有的 object 檔中的 `.text` section 的內容
* 所有的 `.multiboot_header` 的內容必須要加到第一個新產生的 section (`.boot`)，主要確保他們在 executable 的開頭，這是很重要的，因為 GRUB 一開始就會去找檔案中的 Multiboot header。

下一步我們試著建立 ELF object 檔並且用寫好的 linker 腳本 link 他們：

```
> nasm -f elf64 multiboot_header.asm
> nasm -f elf64 boot.asm
> ld -n -o kernel.bin -T linker.ld multiboot_header.o boot.o
```

還有一點很重要的就是要傳 `-n` (或是 `--nmagic`) flag 到 linker，主要是要取消 section 的自動對齊機制，不然 executable 會被 liker 以 page 為單位去對齊 `.boot` section，這會導致 GRUB 找不到 Multiboot header，因為他已經不是最一開始的位置。

我們可以使用 `objdump` 把產生出來的 executable 中的 sections 資訊印出，並且驗證出 `.boot` section 有一個小的 file offset：

```
> objdump -h kernel.bin
kernel.bin:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .boot         00000018  0000000000100000  0000000000100000  00000080  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .text         0000000b  0000000000100020  0000000000100020  000000a0  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
```

注意：`ld` 跟 `objdump` 這兩個命令有平台上的限制。如果你在 `x86_64` 架構下沒辦法使用的話，你必須要自己去 [跨平台編譯這些工具](http://os.phil-opp.com/cross-compile-binutils.html)，接下來再使用編譯出來的 `x86_64-elf-ld` 和 `x86_64-elf-objdump` 來進行。

### Creating the ISO

最後一個步驟就是建立一個用 GRUB 來作為 bootloader 的 ISO image，我們必須建立如下所示的目錄結構，並且放入剛剛編譯好的 `kernel.bin`。

```
isofiles
└── boot
    ├── grub
    │   └── grub.cfg
    └── kernel.bin
```

`grub.cfg` 定義了 kernel 的位置及名稱並且有支援 Multiboot 2。看起來會像這樣：

```
set timeout=0
set default=0

menuentry "my os" {
    multiboot2 /boot/kernel.bin
    boot
}
```

現在就用下面命令來建立可以 boot 的 image：

```
grub-mkrescue -o os.iso isofiles
```

注意：`grub-mkrescue` 會有些平台上的問題。如果不能正常使用時，可以嘗試以下步驟：

* 在執行的時候試著加上 `--verbose`
* 確定 `xorriso` 是不是已經裝好 (`xorriso` 或是 `libisoburn` package 都要確認)
* 如果你是使用 EFI-system，`grub-mkrescue` 預設會去建立一個 EFI image。 你可以代入 `-d /usr/lib/grub/i386-pc` 不去使用 EFI 或是透過安裝 `mtools` package 來得到一個可以使用的 EFI image
* 在某些系統下這個命令的名稱是 `grub2-mkrescue`

### Booting

現在就來 boot 我們所寫的 OS。我們這邊是使用 [QEMU](https://en.wikipedia.org/wiki/QEMU) 模擬器來執行：

```
qemu-system-x86_64 -cdrom os.iso
```

![qemu-ok](qemu-ok.png)

注意到綠色的 `OK` 出現在左上角。如果不成功的話，可以看看下面留言區塊。

現在來總結一下剛剛發生什麼事：

* 一開始 BIOS 會從虛擬的硬碟 (ISO) 去 load bootloader (GRUB)
* bootloader 會去讀 kernel executable 然後找到 Multiboot header
* 接著 bootloader 會把 `.boot` 以及 `.text` sections 複製到記憶體 (位址分別對應到 `0x100000` 和 `0x100020`) (`0x100000` 為 1MB 的位址)
* 最後他就會跳到程式進入點 (`0x100020` 這個值你可以用 `objdump -f` 來看到結果)
* 我們的 kernel 會印出綠色的 `OK` 接著把 CPU 停止

你也可以直接在真的實體機上測試，只要把 ISO 燒到硬碟或是隨身碟上，並且選擇相對應的設備來 boot。

###Build Automation

現在我們每當改一次程式碼，就要下四道指令來執行，這滿麻煩的。所以來寫個 [Makefile](http://mrbook.org/blog/tutorials/make/) 來自動化建置，首先建立空的目錄架構來區分不同架構的程式碼：

```
…
├── Makefile
└── src
    └── arch
        └── x86_64
            ├── multiboot_header.asm
            ├── boot.asm
            ├── linker.ld
            └── grub.cfg
```

Makefile 長的會像下面如此 (但是縮排記得使用 tab 不要用空白)：

```
arch ?= x86_64
kernel := build/kernel-$(arch).bin
iso := build/os-$(arch).iso

linker_script := src/arch/$(arch)/linker.ld
grub_cfg := src/arch/$(arch)/grub.cfg
assembly_source_files := $(wildcard src/arch/$(arch)/*.asm)
assembly_object_files := $(patsubst src/arch/$(arch)/%.asm, \
    build/arch/$(arch)/%.o, $(assembly_source_files))

.PHONY: all clean run iso

all: $(kernel)

clean:
    @rm -r build

run: $(iso)
    @qemu-system-x86_64 -cdrom $(iso)

iso: $(iso)

$(iso): $(kernel) $(grub_cfg)
    @mkdir -p build/isofiles/boot/grub
    @cp $(kernel) build/isofiles/boot/kernel.bin
    @cp $(grub_cfg) build/isofiles/boot/grub
    @grub-mkrescue -o $(iso) build/isofiles 2> /dev/null
    @rm -r build/isofiles

$(kernel): $(assembly_object_files) $(linker_script)
    @ld -n -T $(linker_script) -o $(kernel) $(assembly_object_files)

# compile assembly files
build/arch/$(arch)/%.o: src/arch/$(arch)/%.asm
    @mkdir -p $(shell dirname $@)
    @nasm -felf64 $< -o $@
```

這裡做些註解 (如果你不知道怎麼使用 `make`，那麼你可以去看 [Makefile 的教學](http://mrbook.org/blog/tutorials/make/))：

* `$(wildcard src/arch/$(arch)/*.asm)` 會去選取所有在目錄 `src/arch/$(arch)` 下的 asm 檔，如此一來你每新增一個檔案，你就不用更新 Makefile。
* 透過 `patsubst` 的操作可以設定編譯結果的路徑及檔名，`assembly_object_files` 中也只是把 `src/arch/$(arch)/XYZ.asm` 轉成 `build/arch/$(arch)/XYZ.o` 諸如此類。
* `$<` 和 `$@` 在編譯規則 (assembly target) 裡面是一種自動變數 ([automatic variable](https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html))
* 如果你是使用[跨平台編譯的工具](http://os.phil-opp.com/cross-compile-binutils.html)的話，請把 `ld` 取代成 `x86_64-elf-ld`

```
* $@ 表示規則中的目標檔 (build/arch/$(arch)/XYZ.o)，在多規則中，看哪個檔案符合規則就是那個檔案。
* $< 表示規則中所需第一個檔案 (src/arch/$(arch)/XYZ.asm)，如果是內隱規則 (Implicit Rules)的話 ，那就是規則所需的第一個檔案。
```

現在我們可以使用 `make` 命令，所有有更新過的 asm 檔案都會被編譯和 link。而 `make iso` 命令則是可以建立新的 ISO image。最後在使用 `make run` 啟動 QEMU 來執行結果。

###What's next?

下一篇中我們會建立一個 page table 並且做一些 CPU 設定，讓 CPU 可以轉換成 64-bit 的 [long mode](https://en.wikipedia.org/wiki/Long_mode)。

---
1. 從表格看到公式 `-(magic + architecture + header_length)`，因為 32bit 的 unsigned int 沒有 sign bit 無法表示負數，當透過用 `0x100000000` (= 2^32) 相減來取負號時，由於沒有 sign bit，所以讓數值以正號表示。最後的結果也因為沒有用到額外的 sign bit(s) ，只用到 32bit，也符合編譯器的要求。

2. 我們不想要將 kernel 載入到像是 `0x0` 的位址上，因為在起始位址算起 1MB 大小中，有很多特別用途記憶體區域 (舉例來說 VGA 的 buffer 是放在 `0xb8000`，所以我們可以透過這個位址來在螢幕上顯示 `OK`)。
