---
layout: post
title: rust で組み込み(Cortex-M3)
category: blog
tags: rust embedded
---

この記事では、rust を使ってCortex-Mの上で直接(ベアメタルで)動作するプログラムの作り方を説明する。全部を自前で書くのではなく、STが提供する CubeMX といったツールや HAL を有効活用することを基本方針とする。

## 環境構築

前提とする環境は次のとおり。ホストについては Ubuntu なら apt-get で入る(なんて楽なんだろう)。CubeMX は ST のサイト([http://www.st.com/ja/development-tools/stm32cubemx.html](http://www.st.com/ja/development-tools/stm32cubemx.html)からインストールする。ダウンロードするのにメールアドレスの登録を求められるが、そういった前時代的なものは滅んで欲しいものだ。ターゲットは、日本橋や通販で購入できるだろう。

Host
* OS: Linux(Ubuntu 16.04LTS 64bit)
* compiler: gcc-arm-none-eabi(4.9.3)
* binutils: binutils-arm-none-eabi(2.26)
* newlib: newlib-arm-none-eabi(2.2.0)
* Debugger: gdb-arm-none-eabi(7.10)
* Flasher: Open-OCD(0.9.0)
* CubeMX:4.22.0

Target
* Board: NUCLEO-F103RB([http://www.st.com/ja/evaluation-tools/nucleo-f103rb.html](http://www.st.com/ja/evaluation-tools/nucleo-f103rb.html))
* MCU: STM32F103RB([http://www.st.com/ja/microcontrollers/stm32f103rb.html](http://www.st.com/ja/microcontrollers/stm32f103rb.html))
* Debugger: ST-Link/V2(On board)

ここでターゲットにしているMCUである STM32F103RB は、Cortex-M というカテゴリーに属する。'A','R','M'のなかの最下層の'M'だ。Cortex-M には、さらに 'M0', 'M0+', 'M1', M3', 'M4', 'M7' とランクがわかれている。STM32F103RB は Cortex-M3 に属する。Cortex-M3 は ARM v7-M Thumb 命令セットを持つ。

### xargo

クロスコンパイル環境では`cargo`の代わりに`xargo`というツールを使う。次のコマンドでインストールできる。

```
$ cargo install xargo
```

`xargo`の使い方は`cargo@とほとんど同じだ(というか、ラッパーである)。新しいプロジェクトを作成するには次のようにすればよい。
```
$ xargo new stm32f1_blinky --bin
$ cd stm32f1_blinky
```

### nightly

2017年7月時点では、rust のクロスコンパイル機能は nightly リリースでしか使えない。nightly リリースをインストールする。

```
$ rustup install nightly
```

そして、このディレクトリで nightly を使うようにする。ただし、この設定はディレクトリローカルではなく、`~/.rustup/settings.toml`に書かれることに注意しよう。

```
$ rustup override set nightly
```

次のように、`thumbv7m-none-eabi`がサポートされていることがわかる。

```
$ rustc --version          
rustc 1.20.0-nightly (15aa15b03 2017-07-21)
$ rustc --print target-list
…
thumbv6m-none-eabi
thumbv7em-none-eabi
thumbv7em-none-eabihf
thumbv7m-none-eabi
…
```

ちなみに、この記法は target triple と呼ばれているものである。1番目の項目が命令アーキテクチャ、2番目の項目が OSの種類(noneはOS無し)、3番目の項目が呼び出し規約を表す。

## クロスコンパイル

最小限のプログラムを次に示す。まずは写経してみて欲しい。

```rust
// src/main.rs
#![no_std]
#![no_main]
#![feature(lang_items)]
#![feature(start)]

#[no_mangle]
#[start]
pub extern fn main() {
	loop {}
}

#[lang="panic_fmt"]
pub fn panic_fmt() -> ! { loop {} }

#[lang="eh_personality"]
extern fn eh_personality () {}
```

少しずつ解説していこう。

```rust
#![no_std]
```

このクレート内で標準ライブラリを使わないことを宣言する。`#![...]`記法は、それが書かれているコンテンツに影響を与えることを示している。

```rust
#![no_main]
```

このクレート内で通常のエントリポイントである`fn main`を持たないことを宣言する。

```rust
#![feature(lang_items)]
```
言語拡張である`#[lang=...]`を使えるようにする。
```rust
#![feature(start)]
```
`#![start]`でエントリポイントを指定できるようにする。
```rust
#[no_mangle]
```
次の関数のマングリングをしない。マングリングとは、識別子の名前を改造して型情報などを含ませることを言う。つまり`fn main`は`_main`という名前のままでオブジェクトファイルに含まれ、その名前でリンク可能となる。

```rust
#[start]
```
次の関数をスタートポイントとする。
```rust
pub extern fn main() {
	loop {}
}
```
何もせず、ただ無限ループする関数を作成する。
```rust
#[lang="panic_fmt"]
pub fn panic_fmt() -> ! { loop {} }

#[lang="eh_personality"]
extern fn eh_personality () {}
```
言語が必要としているエラーハンドリング関数を定義する。実体は何もしない。

### リンカスクリプト

組み込みの場合は、メモリアドレスもまちまちなので、リンカスクリプトで教えてやらなければならない。とりあえず`layout.ld`というファイルを作成する。まずは最低限、次の内容があれば、ビルドはできるだろう。MCUが本記事と異なる場合、`RAM`と`FLASH`の`ORIGIN`と`LENGTH`を修正すれば良い。

```
/* Entry Point */
ENTRY(Reset_Handler)

/* Specify the memory areas */
MEMORY
{
    RAM (xrw)   : ORIGIN = 0x20000000, LENGTH = 20K
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 128K
}

/* Define output sections */
SECTIONS
{
    /* The program code and other data goes into FLASH */
    .text :
    {
        *(.text)           /* .text sections (code) */
        *(.text*)          /* .text* sections (code) */
    } >FLASH
}
```
このリンカスクリプトは gcc のリンカ(arm-none-eabi-ld)がわかるように書く。rustとは直接の関係ない。

### .cargo/config

リンクの設定をリンカに渡すために`.cargo/config`というファイルを用意し、次を記述する。
```
[target.thumbv7m-none-eabi]
rustflags = [
    "-C", "link-arg=-Tlayout.ld",
    "-C", "link-arg=-nostartfiles",
]
```
### xargo build

`xargo build --target=thumbv7m-none-eabi`とすれば、ビルドされる。`xargo`は、`libcore`という`libstd`に代わる最低限のライブラリをダウンロードしてリンクしてくれる。`--verbose`をつけると、何が行われているか、少しだけ詳細にわかる。例えば、`libcore`は最適化されてビルドされるが、ユーザーコードはデバッグ情報付きでビルドされる。また、libcore は `xargo` のシステム領域(~/.xargo/lib/rustlib/thumbv7m-none-eabi/lib/)に置かれる。

```
$ xargo build --target=thumbv7m-none-eabi --verbose            
+ "rustc" "--print" "sysroot"
+ "rustc" "--print" "target-list"
+ "cargo" "build" "--release" "--manifest-path" "/tmp/xargo.urp2WMweOjXe/Cargo.toml" "--target" "thumbv7m-none-eabi" "-v" "-p" "core"
   Compiling core v0.0.0 (file://$(HOME)/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/src/libcore)
     Running `rustc --crate-name core $(HOME)/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/src/libcore/lib.rs --crate-type lib --emit=dep-info,link -C opt-level=3 -C metadata=a31897ae4d1ed2cf -C extra-filename=-a31897ae4d1ed2cf --out-dir /tmp/xargo.urp2WMweOjXe/target/thumbv7m-none-eabi/release/deps --target thumbv7m-none-eabi -L dependency=/tmp/xargo.urp2WMweOjXe/target/thumbv7m-none-eabi/release/deps -L dependency=/tmp/xargo.urp2WMweOjXe/target/release/deps -C link-arg=-Tlayout.ld -C link-arg=-nostartfiles --sysroot $(HOME)/.xargo`
    Finished release [optimized] target(s) in 18.84 secs
+ "cargo" "build" "--target=thumbv7m-none-eabi" "--verbose"
   Compiling cross v0.1.0 (file://$(PROJECT_DIR))
     Running `rustc --crate-name cross src/main.rs --crate-type bin --emit=dep-info,link -C debuginfo=2 -C metadata=5992646b6a6bb6fc -C extra-filename=-5992646b6a6bb6fc --out-dir $(PROJECT_DIR)/target/thumbv7m-none-eabi/debug/deps --target thumbv7m-none-eabi -L dependency=$(PROJECT_DIR)/target/thumbv7m-none-eabi/debug/deps -L dependency=$(PROJECT_DIR)/target/debug/deps -C link-arg=-Tlayout.ld -C link-arg=-nostartfiles --sysroot $(HOME)/.xargo`
    Finished dev [unoptimized + debuginfo] target(s) in 0.31 secs
```

## Lチカ

組み込みでの"Hello, World."は、Lチカである。最終的には端子設定は CubeMX を利用することを目標とするが、その結果からカンニングして最小限のプログラムを示す。これは NUCLEO-F103RB に依存している。各種レジスタのアドレスはハードコードされているし、LEDがGPIOのA05につながっていることが前提だ。

### src/main.rs

```rust
#![no_std] // std を使わない。1.6.0以降だと、これで自動的に libcore が使われる。
#![no_main] // rust の標準的な main を使わない
#![feature(lang_items)] // #[lang="..."] を使う宣言。具体的には、下の #[lang="panic_fmt"]
#![feature(start)] // #[start] を使う宣言。

#![feature(asm)]  // asm を使う。
#![feature(core_intrinsics)] // core_intrinsics を使う。
use core::intrinsics::volatile_store; // メモリ直書きには volatile_store を使う。

// レジスタアドレスの定義
pub const PERIPH_BASE: u32      = 0x40000000;

pub const APB2PERIPH_BASE: u32  = PERIPH_BASE + 0x10000;
pub const GPIOA_BASE: u32       = APB2PERIPH_BASE + 0x0800;
pub const CRL_OFFSET: u32       = 0x00;
pub const BSRR_OFFSET: u32      = 0x10;
pub const GPIO_PIN_5: u32       = 5;

pub const AHBPERIPH_BASE: u32   = PERIPH_BASE + 0x20000;
pub const RCC_BASE: u32         = AHBPERIPH_BASE + 0x1000;
pub const CR_OFFSET: u32        = 0x00;
pub const CFGR_OFFSET: u32      = 0x04;
pub const CIR_OFFSET: u32       = 0x08;
pub const APB2ENR_OFFSET: u32   = 0x18;

pub const FLASH_BASE: u32       = 0x08000000;
pub const VECT_TAB_OFFSET: u32  = 0x0;
pub const VTOR_OFFSET: u32      = 8;

pub const SCS_BASE: u32         = 0xE000E000;
pub const SCB_BASE: u32         = SCS_BASE + 0x0D00;

#[no_mangle] // mangling(名前修飾)を使わない。
#[start] // エントリーポイントを指定。
pub extern fn main() {

    let apb2enr = (RCC_BASE+APB2ENR_OFFSET) as *mut u32;
    let crl     = (GPIOA_BASE+CRL_OFFSET) as *mut u32;

    unsafe {
        volatile_store(apb2enr, *apb2enr | (1<<2));
        volatile_store(crl, *crl & (!(6<<20)));
        volatile_store(crl, *crl | (2<<20));
    }

    let bsrr    = (GPIOA_BASE+BSRR_OFFSET)  as *mut u32;

    loop {
        unsafe {
            volatile_store(bsrr, 1 << GPIO_PIN_5);  // 点灯
        }
        for _ in 1..400000 {
            unsafe {asm!("");}
        }

        unsafe {
            volatile_store(bsrr, (1 << GPIO_PIN_5) << 16); // 消灯
        }

        for _ in 1..400000 {
            unsafe {asm!("");}
        }
	}
}

#[no_mangle]
pub extern fn SystemInit(){
    let rcc_cr   = (RCC_BASE+CR_OFFSET) as *mut u32;
    let rcc_cfgr = (RCC_BASE+CFGR_OFFSET) as *mut u32;
    let rcc_cir  = (RCC_BASE+CIR_OFFSET) as *mut u32;
    let scb_vtor = (SCB_BASE+VTOR_OFFSET) as *mut u32;

    unsafe {
        volatile_store(rcc_cr, *rcc_cr | 0x00000001);
        volatile_store(rcc_cfgr, *rcc_cfgr & 0xf0f0000);
        volatile_store(rcc_cr, *rcc_cr & 0xfef6ffff);
        volatile_store(rcc_cr, *rcc_cr & 0xfffbffff);
        volatile_store(rcc_cfgr, *rcc_cfgr & 0xff80ffff);
        volatile_store(rcc_cir, 0x009f0000);
        volatile_store(scb_vtor, FLASH_BASE | VECT_TAB_OFFSET);
    }
}

#[lang="panic_fmt"] // コンパイラの失敗メカニズムのために必要な関数
pub fn panic_fmt(_fmt: &core::fmt::Arguments, _file_line: &(&'static str, usize)) -> ! {
	loop {}
}

#[lang="eh_personality"] // コンパイラの失敗メカニズムのために必要な関数
extern fn eh_personality (){}
```

順に解説してゆこう。

```rust
#[no_mangle] // mangling(名前修飾)を使わない。
#[start] // エントリーポイントを指定。
pub extern fn main() {
```
main は、アセンブラのスタートアップから呼ばれる。`_main`というシンボルで呼ばれたいので、`#[no_mangle]`指定をしてある。

```rust
    let apb2enr = (RCC_BASE+APB2ENR_OFFSET) as *mut u32;
    let crl     = (GPIOA_BASE+CRL_OFFSET) as *mut u32;
```                
レジスタのアドレスの型は `as *mut u32`というようにキャストする。

```rust
    unsafe {
        volatile_store(apb2enr, *apb2enr | (1<<2));
        volatile_store(crl, *crl & (!(6<<20)));
        volatile_store(crl, *crl | (2<<20));
    }
```                
レジスタへの代入は `volatile_store` を使う。そして、これは危険な操作なので `unsafe`で囲う。

```rust
    let bsrr    = (GPIOA_BASE+BSRR_OFFSET)  as *mut u32;
        unsafe {
            volatile_store(bsrr, 1 << GPIO_PIN_5);  // 点灯
        }
```
GPIOAのBSRRの下半分のアドレスに1を書くと、対応するポートがHになる。

```rust
        for _ in 1..400000 {
            unsafe {asm!("");}
        }
```
ビジーループで待つ。


```rust
            volatile_store(bsrr, (1 << GPIO_PIN_5) << 16); // 消灯
```
GPIOAのBSRRの上半分のアドレスに1を書くと、対応するポートがLになる。

```rust
#[no_mangle]
pub extern fn SystemInit(){
```
`SystemInit`もスタートアップから呼ばれる別のルーチン。main より前に呼ばれて、クロックの設定をする。

# src/layout.ld

```
/* Entry Point */
ENTRY(Reset_Handler)

/* Highest address of the user mode stack */
_estack = 0x20005000;    /* end of RAM */

/* Specify the memory areas */
MEMORY
{
  RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 20K
  FLASH (rx)      : ORIGIN = 0x8000000, LENGTH = 128K
}

/* Define output sections */
SECTIONS
{
  . = ALIGN(4);
  /* The startup code goes first into FLASH */
  .isr_vector :
  {
    KEEP(*(.isr_vector)) /* Startup code */
  } >FLASH

  /* The program code and other data goes into FLASH */
  .text :
  {
    *(.text)           /* .text sections (code) */
    *(.text*)          /* .text* sections (code) */
    KEEP (*(.init))
  } >FLASH

  /* used by the startup to initialize data */
  _sidata = LOADADDR(.data);

  /* Initialized data sections goes into RAM, load LMA copy after code */
  .data : 
  {
    _sdata = .;        /* create a global symbol at data start */
    _edata = .;        /* define a global symbol at data end */
  } >RAM AT> FLASH
  
  /* Uninitialized data section */
  .bss :
  {
    /* This is used by the startup in order to initialize the .bss secion */
    _sbss = .;         /* define a global symbol at bss start */
    _ebss = .;         /* define a global symbol at bss end */
  } >RAM

  /* Remove information from the standard libraries */
  /DISCARD/ :
  {
    libc.a ( * )
    libm.a ( * )
    libgcc.a ( * )
  }

}
```

_estack と RAM, FLASH のアドレス、大きさはチップ固有だが、詰め方のところはたいがい同じで行けるだろう。

### .cargo/config

ちゃんと動かすためには、修正が必要だ。
```
[build]
target = "thumbv7m-none-eabi"

[target.thumbv7m-none-eabi]
rustflags = [
    "-Z", "no-landing-pads",
    "-C", "opt-level=2",
    "-C", "link-arg=-mcpu=cortex-m3",
    "-C", "link-arg=-mthumb",
    "-C", "link-arg=-mfloat-abi=soft",
    "-C", "link-arg=-specs=nosys.specs",
    "-C", "link-arg=-specs=nano.specs",
    "-C", "link-arg=-Tsrc/layout.ld"
]
```

```
[build]
target = "thumbv7m-none-eabi"
```
ここに指定しておくと、xargo build 時に `--target=...` を一々指定しなくても良い。

```
    "-Z", "no-landing-pads",
    "-C", "opt-level=2",
```
この辺は、ノウハウであり呪文。特に、`opt-level=2`は、最適化しないとエラーになることがあるので必須。

* `-Z no-landing-pads`: debug option -- omit landing pads for unwinding
* `-C opt-level=2`: codegen option -- optimize with possible levels 0-3, s, or z
```
    "-C", "link-arg=-mcpu=cortex-m3",
    "-C", "link-arg=-mthumb",
    "-C", "link-arg=-mfloat-abi=soft",
    "-C", "link-arg=-specs=nosys.specs",
    "-C", "link-arg=-specs=nano.specs",
    "-C", "link-arg=-Tsrc/layout.ld"
```
リンカに渡すオプション。大体 gcc の時と同じ。

### src/startup_stm32f103xb.s
```
  .syntax unified
  .cpu cortex-m3
  .fpu softvfp
  .thumb

.global g_pfnVectors
.global Default_Handler

.equ  BootRAM, 0xF108F85F
/**
 * @brief  This is the code that gets called when the processor first
 *          starts execution following a reset event. Only the absolutely
 *          necessary set is performed, after which the application
 *          supplied main() routine is called.
 * @param  None
 * @retval : None
*/

  .section .text.Reset_Handler
  .weak Reset_Handler
  .type Reset_Handler, %function
Reset_Handler:

/* Copy the data segment initializers from flash to SRAM */
  movs r1, #0
  b LoopCopyDataInit

CopyDataInit:
  ldr r3, =_sidata
  ldr r3, [r3, r1]
  str r3, [r0, r1]
  adds r1, r1, #4

LoopCopyDataInit:
  ldr r0, =_sdata
  ldr r3, =_edata
  adds r2, r0, r1
  cmp r2, r3
  bcc CopyDataInit
  ldr r2, =_sbss
  b LoopFillZerobss
/* Zero fill the bss segment. */
FillZerobss:
  movs r3, #0
  str r3, [r2], #4

LoopFillZerobss:
  ldr r3, = _ebss
  cmp r2, r3
  bcc FillZerobss

/* Call the clock system intitialization function.*/
    bl  SystemInit
/* Call static constructors */
    bl __libc_init_array
/* Call the application's entry point.*/
  bl main
  bx lr
.size Reset_Handler, .-Reset_Handler

/**
 * @brief  This is the code that gets called when the processor receives an
 *         unexpected interrupt.  This simply enters an infinite loop, preserving
 *         the system state for examination by a debugger.
 *
 * @param  None
 * @retval : None
*/
    .section .text.Default_Handler,"ax",%progbits
Default_Handler:
Infinite_Loop:
  b Infinite_Loop
  .size Default_Handler, .-Default_Handler
/******************************************************************************
*
* The minimal vector table for a Cortex M3.  Note that the proper constructs
* must be placed on this to ensure that it ends up at physical address
* 0x0000.0000.
*
******************************************************************************/
  .section .isr_vector,"a",%progbits
  .type g_pfnVectors, %object
  .size g_pfnVectors, .-g_pfnVectors


g_pfnVectors:

  .word _estack
  .word Reset_Handler
  .word NMI_Handler
  .word HardFault_Handler
  .word MemManage_Handler
  .word BusFault_Handler
  .word UsageFault_Handler
  .word 0
  .word 0
  .word 0
  .word 0
  .word SVC_Handler
  .word DebugMon_Handler
  .word 0
  .word PendSV_Handler
  .word SysTick_Handler
  .word WWDG_IRQHandler
  .word PVD_IRQHandler
  .word TAMPER_IRQHandler
  .word RTC_IRQHandler
  .word FLASH_IRQHandler
  .word RCC_IRQHandler
  .word EXTI0_IRQHandler
  .word EXTI1_IRQHandler
  .word EXTI2_IRQHandler
  .word EXTI3_IRQHandler
  .word EXTI4_IRQHandler
  .word DMA1_Channel1_IRQHandler
  .word DMA1_Channel2_IRQHandler
  .word DMA1_Channel3_IRQHandler
  .word DMA1_Channel4_IRQHandler
  .word DMA1_Channel5_IRQHandler
  .word DMA1_Channel6_IRQHandler
  .word DMA1_Channel7_IRQHandler
  .word ADC1_2_IRQHandler
  .word USB_HP_CAN1_TX_IRQHandler
  .word USB_LP_CAN1_RX0_IRQHandler
  .word CAN1_RX1_IRQHandler
  .word CAN1_SCE_IRQHandler
  .word EXTI9_5_IRQHandler
  .word TIM1_BRK_IRQHandler
  .word TIM1_UP_IRQHandler
  .word TIM1_TRG_COM_IRQHandler
  .word TIM1_CC_IRQHandler
  .word TIM2_IRQHandler
  .word TIM3_IRQHandler
  .word TIM4_IRQHandler
  .word I2C1_EV_IRQHandler
  .word I2C1_ER_IRQHandler
  .word I2C2_EV_IRQHandler
  .word I2C2_ER_IRQHandler
  .word SPI1_IRQHandler
  .word SPI2_IRQHandler
  .word USART1_IRQHandler
  .word USART2_IRQHandler
  .word USART3_IRQHandler
  .word EXTI15_10_IRQHandler
  .word RTC_Alarm_IRQHandler
  .word USBWakeUp_IRQHandler
  .word 0
  .word 0
  .word 0
  .word 0
  .word 0
  .word 0
  .word 0
  .word BootRAM          /* @0x108. This is for boot in RAM mode for
                            STM32F10x Medium Density devices. */

/*******************************************************************************
*
* Provide weak aliases for each Exception handler to the Default_Handler.
* As they are weak aliases, any function with the same name will override
* this definition.
*
*******************************************************************************/

  .weak NMI_Handler
  .thumb_set NMI_Handler,Default_Handler
  .weak HardFault_Handler
  .thumb_set HardFault_Handler,Default_Handler
  .weak MemManage_Handler
  .thumb_set MemManage_Handler,Default_Handler
  .weak BusFault_Handler
  .thumb_set BusFault_Handler,Default_Handler
  .weak UsageFault_Handler
  .thumb_set UsageFault_Handler,Default_Handler
  .weak SVC_Handler
  .thumb_set SVC_Handler,Default_Handler
  .weak DebugMon_Handler
  .thumb_set DebugMon_Handler,Default_Handler
  .weak PendSV_Handler
  .thumb_set PendSV_Handler,Default_Handler
  .weak SysTick_Handler
  .thumb_set SysTick_Handler,Default_Handler
  .weak WWDG_IRQHandler
  .thumb_set WWDG_IRQHandler,Default_Handler
  .weak PVD_IRQHandler
  .thumb_set PVD_IRQHandler,Default_Handler
  .weak TAMPER_IRQHandler
  .thumb_set TAMPER_IRQHandler,Default_Handler
  .weak RTC_IRQHandler
  .thumb_set RTC_IRQHandler,Default_Handler
  .weak FLASH_IRQHandler
  .thumb_set FLASH_IRQHandler,Default_Handler
  .weak RCC_IRQHandler
  .thumb_set RCC_IRQHandler,Default_Handler
  .weak EXTI0_IRQHandler
  .thumb_set EXTI0_IRQHandler,Default_Handler
  .weak EXTI1_IRQHandler
  .thumb_set EXTI1_IRQHandler,Default_Handler
  .weak EXTI2_IRQHandler
  .thumb_set EXTI2_IRQHandler,Default_Handler
  .weak EXTI3_IRQHandler
  .thumb_set EXTI3_IRQHandler,Default_Handler
  .weak EXTI4_IRQHandler
  .thumb_set EXTI4_IRQHandler,Default_Handler
  .weak DMA1_Channel1_IRQHandler
  .thumb_set DMA1_Channel1_IRQHandler,Default_Handler
  .weak DMA1_Channel2_IRQHandler
  .thumb_set DMA1_Channel2_IRQHandler,Default_Handler
  .weak DMA1_Channel3_IRQHandler
  .thumb_set DMA1_Channel3_IRQHandler,Default_Handler
  .weak DMA1_Channel4_IRQHandler
  .thumb_set DMA1_Channel4_IRQHandler,Default_Handler
  .weak DMA1_Channel5_IRQHandler
  .thumb_set DMA1_Channel5_IRQHandler,Default_Handler
  .weak DMA1_Channel6_IRQHandler
  .thumb_set DMA1_Channel6_IRQHandler,Default_Handler
  .weak DMA1_Channel7_IRQHandler
  .thumb_set DMA1_Channel7_IRQHandler,Default_Handler
  .weak ADC1_2_IRQHandler
  .thumb_set ADC1_2_IRQHandler,Default_Handler
  .weak USB_HP_CAN1_TX_IRQHandler
  .thumb_set USB_HP_CAN1_TX_IRQHandler,Default_Handler
  .weak USB_LP_CAN1_RX0_IRQHandler
  .thumb_set USB_LP_CAN1_RX0_IRQHandler,Default_Handler
  .weak CAN1_RX1_IRQHandler
  .thumb_set CAN1_RX1_IRQHandler,Default_Handler
  .weak CAN1_SCE_IRQHandler
  .thumb_set CAN1_SCE_IRQHandler,Default_Handler
  .weak EXTI9_5_IRQHandler
  .thumb_set EXTI9_5_IRQHandler,Default_Handler
  .weak TIM1_BRK_IRQHandler
  .thumb_set TIM1_BRK_IRQHandler,Default_Handler
  .weak TIM1_UP_IRQHandler
  .thumb_set TIM1_UP_IRQHandler,Default_Handler
  .weak TIM1_TRG_COM_IRQHandler
  .thumb_set TIM1_TRG_COM_IRQHandler,Default_Handler
  .weak TIM1_CC_IRQHandler
  .thumb_set TIM1_CC_IRQHandler,Default_Handler
  .weak TIM2_IRQHandler
  .thumb_set TIM2_IRQHandler,Default_Handler
  .weak TIM3_IRQHandler
  .thumb_set TIM3_IRQHandler,Default_Handler
  .weak TIM4_IRQHandler
  .thumb_set TIM4_IRQHandler,Default_Handler
  .weak I2C1_EV_IRQHandler
  .thumb_set I2C1_EV_IRQHandler,Default_Handler
  .weak I2C1_ER_IRQHandler
  .thumb_set I2C1_ER_IRQHandler,Default_Handler
  .weak I2C2_EV_IRQHandler
  .thumb_set I2C2_EV_IRQHandler,Default_Handler
  .weak I2C2_ER_IRQHandler
  .thumb_set I2C2_ER_IRQHandler,Default_Handler
  .weak SPI1_IRQHandler
  .thumb_set SPI1_IRQHandler,Default_Handler
  .weak SPI2_IRQHandler
  .thumb_set SPI2_IRQHandler,Default_Handler
  .weak USART1_IRQHandler
  .thumb_set USART1_IRQHandler,Default_Handler
  .weak USART2_IRQHandler
  .thumb_set USART2_IRQHandler,Default_Handler
  .weak USART3_IRQHandler
  .thumb_set USART3_IRQHandler,Default_Handler
  .weak EXTI15_10_IRQHandler
  .thumb_set EXTI15_10_IRQHandler,Default_Handler
  .weak RTC_Alarm_IRQHandler
  .thumb_set RTC_Alarm_IRQHandler,Default_Handler
  .weak USBWakeUp_IRQHandler
  .thumb_set USBWakeUp_IRQHandler,Default_Handler
```

`Reset_Handler` から初期化をして、SystemInit, main を読んでいることがわかれば、とりあえずは十分だろう。あとは、すべての割り込みハンドラに `Default_Handler` をセットしている。

### build.rs

rust のコードだけでなくアセンブラのコードもコンパイルしてリンクしなければならない。cargo にビルド手順を指示するためには、build.rs を書いて、ビルドの手順を指定する。cargo build すると、build.rs がコンパイル＆実行されて、その出力にしたがってビルドが行われる。

```rust
use std::process::Command;
use std::env;
use std::path::Path;

fn main() {
    let out_dir = env::var("OUT_DIR").unwrap();

    let mut objs: Vec<String> = Vec::new();

    Command::new("arm-none-eabi-as")
        .args(&["-mcpu=cortex-m3", "-mthumb", "-mfloat-abi=soft"])
        .args(&["src/startup_stm32f103xb.s"])
        .args(&["-o"])
        .arg(&format!("{}/startup_stm32f103xb.o", out_dir))
        .status().unwrap();

    Command::new("arm-none-eabi-ar")
        .args(&["crus", "libcube.a"])
        .arg(&format!("{}/startup_stm32f103xb.o", out_dir))
        .current_dir(&Path::new(&out_dir))
        .status().unwrap();

    println!("cargo:rustc-link-search=native={}", out_dir);
    println!("cargo:rustc-link-lib=static=cube");

    println!("cargo:rerun-if-changed=build.rs");
    println!("cargo:rerun-if-changed=src/layout.ld");
    println!("cargo:rerun-if-changed=src/startup_stm32f103xb.s");
}
```

逐一、解説してゆこう。

```rust
    let out_dir = env::var("OUT_DIR").unwrap();
```
環境変数から出力ディレクトリを取り出す。

```rust
    Command::new("arm-none-eabi-as")
        .args(&["-mcpu=cortex-m3", "-mthumb", "-mfloat-abi=soft"])
        .args(&["src/startup_stm32f103xb.s"])
        .args(&["-o"])
        .arg(&format!("{}/startup_stm32f103xb.o", out_dir))
        .status().unwrap();
```
アセンブラを実行してオブジェクトファイルを作成する。

```rust
    Command::new("arm-none-eabi-ar")
        .args(&["crus", "libcube.a"])
        .arg(&format!("{}/startup_stm32f103xb.o", out_dir))
        .current_dir(&Path::new(&out_dir))
        .status().unwrap();
```
ar を実行してライブラリを作成する。

```rust
    println!("cargo:rustc-link-search=native={}", out_dir);
    println!("cargo:rustc-link-lib=static=cube");
```
ライブラリをリンクするように rust に指示する。

```rust
    println!("cargo:rerun-if-changed=build.rs");
    println!("cargo:rerun-if-changed=src/layout.ld");
    println!("cargo:rerun-if-changed=src/startup_stm32f103xb.s");
```
変更があった場合は再ビルドするように指示する。

### flash.sh

出来上がったファイルは Open-OCDを使ってターゲットに書き込むが、次のようなシェルスクリプトがあると便利だ。ポートを開くためにはルート権限が必要になる。

```
#!/bin/sh

DIR=target/thumbv7m-none-eabi/debug
PROJECT=blinky
ELF_FILE=${DIR}/${PROJECT}
echo "Write $ELF_FILE"
openocd -f board/st_nucleo_f103rb.cfg -c "init" -c "reset init" -c "stm32f1x mass_erase 0" -c "flash write_image $ELF_FILE" -c "reset halt" -c "reset run" -c "exit"
```

<div align="center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/ohu4herKoas" frameborder="0" allowfullscreen></iframe></div>

## CubeMXを使う

組み込み用マイコンはいろいろな周辺機器を内蔵していて「ペリフェラル」と呼ばれる。どのようなペリフェラルを内蔵しているかはマイコンごとの特徴であり、チップ毎に異なる。データシートを見ながらアプリごとに設定をレジスタにか来ま無くてはならない。

CubeMX は、GUIでピン配置やペリフェラルの設定をして、そういったコードを生成してくれるツールだ。低レベルデバイスドライバ(Hardware Abstraction Layer:HAL)ライブラリも同梱されている。CubeMXはST社のツールだが、他のメーカーでも似たようなツールがあることが多い。

組み込みの世界ではCが標準的に使われており、CubeMXの生成コードもCだ。rustのFFI機能を使ってCubeMXのCコードをリンクする。HALやペリフェラル設定をrustで手書き、または民間ライブラリをダウンロードしてきても良いが、ここでは前者の方法を採用する。

### 設計方針

次のように設計方針を定める。

* HALの部分は、STのFirmware Packageをリンクする。
* HALを rust でラップし、使いやすい API を提供する。
* ペリフェラルの設定は、CubeMXで生成したものをリンクする。
* CubeMXで生成した main() でペリフェラルの初期化まで行ってから、main.rs の rust_main を呼び、rust に制御を渡す。

### コード生成

CubeMXをダウンロードして起動する。ダウンロードにはメールアドレスの登録が必要だ。CubeMXはJavaベースのツールなのでLinuxでも普通に使える。

起動したら、「New Project」からプロジェクトを新規作成し、ボードを選択する。今回は NUCLEO-F103RB だ。ボードを選択すると、ボードで接続されているI/Oに合わせて、ピン設定がとペリフェラル設定がロードされる。自作のボードの場合は、MCUを選択して手動でピン設定する。

<div align="center"><img src="/images/2017-07-30-cubemx1.png" alt="cubemx"></div>

メニューの Project → Settings で、<strong>Toolchain/IDE を SW4STM32 を選択する</strong>のがコツ。ほかは適切に選択すれば良い。Project → Generate Code で必要な Firmware Package がインポートされ、初期化コードが生成される。
                
次のようになっているはずだ。

```
$ tree
.
├── Cargo.toml                           # cargo new で生成された
└── src
    ├── cubemx
    │   ├── Drivers                     # HAL Driverのヘッダとコード。変更不要。
    │   │   ├── CMSIS
    │   │   │   ├── Device
    │   │   │   │   └── ST
    │   │   │   │       └── STM32F1xx
    │   │   │   │           ├── Include       # -I で指定する。
    │   │   │   │           │   ├── stm32f103xb.h
    │   │   │   │           │   ├── stm32f1xx.h
    │   │   │   │           │   └── system_stm32f1xx.h
    │   │   │   │           └── Source
    │   │   │   │               └── Templates
    │   │   │   │                   └── gcc
    │   │   │   └── Include                    # -I で指定する。
    │   │   │       ├── core_cm3.h
    │   │   │       └── (略)
    │   │   └── STM32F1xx_HAL_Driver
    │   │       ├── Inc                         # -I で指定する。
    │   │       │   ├── Legacy
    │   │       │   │   └── stm32_hal_legacy.h
    │   │       │   ├── stm32f1xx_hal.h
    │   │       │   ├── stm32f1xx_hal_cortex.h
    │   │       │   └── (略)
    │   │       └── Src
    │   │           ├── stm32f1xx_hal.c
    │   │           └── (略)
    │   ├── Inc                         # CubeMX が生成したコード。ボード固有設定が含まれる。
    │   │   │                                     # -I で指定する。
    │   │   ├── main.h
    │   │   ├── stm32f1xx_hal_conf.h
    │   │   └── stm32f1xx_it.h
    │   ├── STM32F103RBTx_FLASH.ld      # リンカスクリプト
    │   ├── Src                         # CubeMX が生成したコード。ボード固有設定が含まれる。
    │   │   │                            # 必要に応じて修正する。
    │   │   ├── main.c
    │   │   ├── stm32f1xx_hal_msp.c
    │   │   ├── stm32f1xx_it.c
    │   │   └── system_stm32f1xx.c
    │   ├── cubemx.ioc                  # CubeMXのプロジェクトファイル
    │   └── startup
    │       └── startup_stm32f103xb.s   # スタートアップコード
    └── main.rs
```

これまで同様、必要なファイルを作成してゆく。

### .cargo/config

リンカスクリプトはCubeMXか生成したものを指定する。

### src/gpio.rs

Firmware Package の stm32f1xx_hal_gpio.c に対応する API を定義する。
```rust
#![allow(non_snake_case)]

//! Interface of stm32f1xx_hal_gpio.c
//! # Examples
//! ```
//! let io = GPIOA().ReadPin(PIN_5); // gpio::Level
//! GPIOA().WritePin(PIN_5, gpio::Level::High);  // or gpio::Level::Low
//! ```

// レジスタアドレスの定義
const PERIPH_BASE: u32 = 0x40000000;

const APB2PERIPH_BASE: u32 = PERIPH_BASE + 0x10000;
const GPIOA_BASE: u32 = APB2PERIPH_BASE + 0x0800;
const GPIOB_BASE: u32 = APB2PERIPH_BASE + 0x0C00;
const GPIOC_BASE: u32 = APB2PERIPH_BASE + 0x1000;
const GPIOD_BASE: u32 = APB2PERIPH_BASE + 0x1400;
const GPIOE_BASE: u32 = APB2PERIPH_BASE + 0x1800;

pub const PIN_0: u16 = 0x0001;
pub const PIN_1: u16 = 0x0002;
pub const PIN_2: u16 = 0x0004;
pub const PIN_3: u16 = 0x0008;
pub const PIN_4: u16 = 0x0010;
pub const PIN_5: u16 = 0x0020;
pub const PIN_6: u16 = 0x0040;
pub const PIN_7: u16 = 0x0080;
pub const PIN_8: u16 = 0x0100;
pub const PIN_9: u16 = 0x0200;
pub const PIN_10: u16 = 0x0400;
pub const PIN_11: u16 = 0x0800;
pub const PIN_12: u16 = 0x1000;
pub const PIN_13: u16 = 0x2000;
pub const PIN_14: u16 = 0x4000;
pub const PIN_15: u16 = 0x8000;

pub const MODE_INPUT: u32 = 0x00000000;
pub const MODE_OUTPUT_PP: u32 = 0x00000001;
pub const MODE_OUTPUT_OD: u32 = 0x00000011;
pub const MODE_AF_PP: u32 = 0x00000002;
pub const MODE_AF_OD: u32 = 0x00000012;
pub const MODE_AF_INPUT: u32 = MODE_INPUT;
pub const MODE_ANALOG: u32 = 0x00000003;
pub const MODE_IT_RISING: u32 = 0x10110000;
pub const MODE_IT_FALLING: u32 = 0x10210000;
pub const MODE_IT_RISING_FALLING: u32 = 0x10310000;
pub const MODE_EVT_RISING: u32 = 0x10120000;
pub const MODE_EVT_FALLING: u32 = 0x10220000;
pub const MODE_EVT_RISING_FALLING: u32 = 0x10320000;

pub const SPEED_FREQ_LOW: u32 = 0x00000002;
pub const SPEED_FREQ_MEDIUM: u32 = 0x00000001;
pub const SPEED_FREQ_HIGH: u32 = 0x00000003;

pub const NOPULL: u32 = 0x00000000;
pub const PULLUP: u32 = 0x00000001;
pub const PULLDOWN: u32 = 0x00000002;

pub enum Level {
    Low,
    High,
}

#[repr(C)] // C の struct のインポート
pub struct Init {
    pub Pin: u32,
    pub Mode: u32,
    pub Pull: u32,
    pub Speed: u32,
}

#[repr(C)]
pub struct Regs {
    CRL: u32,
    CRH: u32,
    IDR: u32,
    ODR: u32,
    BSRR: u32,
    BRR: u32,
    LCKR: u32,
}

extern "C" {
    pub fn HAL_GPIO_Init(GPIOx: &mut Regs, GPIO_Init: &Init);
    pub fn HAL_GPIO_WritePin(GPIOx: &mut Regs, GPIO_Pin: u16, PinState: u32);
    pub fn HAL_GPIO_ReadPin(GPIOx: &mut Regs, GPIO_Pin: u16) -> u32;
}

impl Regs {
    pub fn Init(&mut self, GPIO_Init: &Init) -> () {
        unsafe {
            HAL_GPIO_Init(self, GPIO_Init);
        }
    }

    pub fn WritePin(&mut self, GPIO_Pin: u16, PinState: Level) -> () {
        match PinState {
            Level::Low => unsafe {
                HAL_GPIO_WritePin(self, GPIO_Pin, 0 as u32);
            },
            Level::High => unsafe {
                HAL_GPIO_WritePin(self, GPIO_Pin, 1 as u32);
            },
        }
    }

    pub fn ReadPin(&mut self, GPIO_Pin: u16) -> Level {
        let ret: u32;
        unsafe {
            ret = HAL_GPIO_ReadPin(self, GPIO_Pin);
        }
        match ret {
            0 => Level::Low,
            _ => Level::High,
        }
    }
}

pub fn GPIOA() -> &'static mut Regs {
    unsafe { &mut *(GPIOA_BASE as *mut Regs) }
}
pub fn GPIOB() -> &'static mut Regs {
    unsafe { &mut *(GPIOB_BASE as *mut Regs) }
}
pub fn GPIOC() -> &'static mut Regs {
    unsafe { &mut *(GPIOC_BASE as *mut Regs) }
}
pub fn GPIOD() -> &'static mut Regs {
    unsafe { &mut *(GPIOD_BASE as *mut Regs) }
}
pub fn GPIOE() -> &'static mut Regs {
    unsafe { &mut *(GPIOE_BASE as *mut Regs) }
}
```

逐次、解説してゆこう。

```rust
#![allow(non_snake_case)]
```
<a href="https://aturon.github.io/README.html">rust のスタイルガイド</a>で、関数名、変数名はスネークケース、型名はキャメルケース、定数は大文字となっている。元のAPIと整合を取るために、このスタイルガイドに反してしまう。警告がいっぱい出るので抑制する。

```rust
//! Interface of stm32f1xx_hal_gpio.c
//! # Examples
//! ```
//! let io = GPIOA().ReadPin(PIN_5); // gpio::Level
//! GPIOA().WritePin(PIN_5, gpio::Level::High);  // or gpio::Level::Low
//! ```
```
`//!`ではじまる行コメントは、(Doxygen のような感じで)ドキュメント化される。書き方は(当然)Markdownだ。`# Example`のような標準的なセクションも定められている。

rust には、スタイルやコメント、テスト、ディレクトリツリー構造など、細部にわたって規約が存在する。そのために、誰が書いてもほとんど同じようになり、他人のコードでも理解しやすい。また、それらの規約は、現代的なプラクティスに基づいている。規約がほとんど無い、Cのような他言語に行った時でも、rust の規約はよいプラクティスとして役に立つはずだ。

```rust
// レジスタアドレスの定義
const PERIPH_BASE: u32 = 0x40000000;

const APB2PERIPH_BASE: u32 = PERIPH_BASE + 0x10000;
const GPIOA_BASE: u32 = APB2PERIPH_BASE + 0x0800;
const GPIOB_BASE: u32 = APB2PERIPH_BASE + 0x0C00;
…(略)
```
関連するレジスタのアドレスを定義する。元ネタは `Drivers/CMSIS/Device/ST/STM32F1xx/Include/stm32f103xb.h`あたり。モジュールの外に公開しなくても良いのて`pub`がついていない。

```rust
pub const PIN_0: u16 = 0x0001;
pub const PIN_1: u16 = 0x0002;
pub const PIN_2: u16 = 0x0004;
…(略)

pub const MODE_INPUT: u32 = 0x00000000;
pub const MODE_OUTPUT_PP: u32 = 0x00000001;
pub const MODE_OUTPUT_OD: u32 = 0x00000011;
…(略)
```
GPIO固有の定義は `Drivers/STM32F1xx_HAL_Driver/Inc/stm32f1xx_hal_gpio.h`あたりにある。ピン番号は公開しなければならない情報なので`pub`を付ける。`MODE_INPUT`などの設定用定数は初期化時に使われ、初期化はCubeMX内で行われる。そのため、通常はモジュール外では使わないが、アプリからポートを再設定するときなどのために公開しておく。

これらの定数は、Cでは`#define GPIO_PIN_0`のようにプレフィックス付きの名前で定義されていた。rust では、モジュール階層に対応した名前空間があり、`gpio::PIN_0`と参照される。冗長さを避けるために、識別子名にはプレフィックスを付けないようにした。

```rust
pub enum Level {
    Low,
    High,
}
```
ポートのH/Lを表す定数。`const`ではなく `enum`で定義しておくことで、必ず `Low`か`High`であることを表現できる。それは人間へ向けての表現だけでなく、コンパイラもそれを理解し、`match`で分岐するときにフルケースであることを強要する。

```rust
#[repr(C)] // C の struct のインポート
pub struct Init {
    pub Pin: u32,
    pub Mode: u32,
    pub Pull: u32,
    pub Speed: u32,
}
```
Cの `typedef struct GPIO_InitTypeDef`に対応する構造体。ペリフェラルの初期化時に使われる。`#[repr(C)]`として、Cの構造体と同じメモリ配列にすることを指示する。同じメモリ配置にすることで、rust での構造体をそのままCのAPIに渡すことができる。

```rust
#[repr(C)]
pub struct Regs {
    CRL: u32,
    CRH: u32,
    IDR: u32,
    ODR: u32,
    BSRR: u32,
    BRR: u32,
    LCKR: u32,
}
```
Cの `typedef struct GPIO_TypeDef`に対応する構造体。MCUにはGPIOのポートごとに、この並びでレジスタが存在する。アドレスを、実際のレジスタに合わせて構造体変数を定義する。そして、その構造体変数を読み書きすることで、ペリフェラルのレジスタを読み書きする。

```rust
pub fn GPIOA() -> &'static mut Regs {
    unsafe { &mut *(GPIOA_BASE as *mut Regs) }
}
```
GPIOA_BASEにはペリフェラルレジスタ群のベースレジスタのアドレスが定義されており、それを`struct Regs`のポインタ(*mut Regs)にキャストして、変更可能な参照(&mut)として返す関数を定義している。生ポインタのキャストは unsafe な操作なので`unsafe`で囲う必要がある。

```rust
impl Regs {
    pub fn WritePin(&mut self, GPIO_Pin: u16, PinState: Level) -> () {
        match PinState {
            Level::Low => unsafe {
                HAL_GPIO_WritePin(self, GPIO_Pin, 0 as u32);
            },
            Level::High => unsafe {
                HAL_GPIO_WritePin(self, GPIO_Pin, 1 as u32);
            },
        }
    }

    pub fn ReadPin(&mut self, GPIO_Pin: u16) -> Level {
        let ret: u32;
        unsafe {
            ret = HAL_GPIO_ReadPin(self, GPIO_Pin);
        }
        match ret {
            0 => Level::Low,
            _ => Level::High,
        }
    }
}
```
`struct Regs`に対して実装を定義する。対応するCのAPIは `void HAL_GPIO_WritePin(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, GPIO_PinState PinState)
`のようになっていて、`GPIO_TypeDef* GPIOx`は、rust では `&mut self`となる。

rust のAPIからCのAPIに変換しているだけだが、CのAPIコールは`unsafe`となる。

また、上述のように`PinState`は`enum Level`なので、`match`で分岐する。

```rust
extern "C" {
    pub fn HAL_GPIO_Init(GPIOx: &mut Regs, GPIO_Init: &Init);
    pub fn HAL_GPIO_WritePin(GPIOx: &mut Regs, GPIO_Pin: u16, PinState: u32);
    pub fn HAL_GPIO_ReadPin(GPIOx: &mut Regs, GPIO_Pin: u16) -> u32;
}
```
CのAPIを外部参照している。引数の型は rust 流に翻訳する必要がある。

### src/main.rs

main.rs の方は gpio.rs のAPIを呼ぶだけで非常にシンプルである。

```rust
#![no_std]
#![no_main]
#![feature(lang_items)]
#![feature(asm)]

mod gpio;
use gpio::{GPIOA, PIN_5, Level};

#[no_mangle]
pub extern fn rust_main() {
    loop {
        GPIOA().WritePin(PIN_5, Level::High);
        for _ in 1..4000000 {
            unsafe {
                asm!("");
            }
        }

        GPIOA().WritePin(PIN_5, Level::Low);
        for _ in 1..4000000 {
            unsafe {
                asm!("");
            }
        }
    }
}

#[lang="panic_fmt"]
pub fn panic_fmt() -> ! {
    loop {}
}

#[lang="eh_personality"]
extern "C" fn eh_personality() {}
```


```rust
mod gpio;
```
main.rs と同じディレクトリ階層にある gpio.rs を使う宣言。これで、ビルド、リンクまで面倒を見てくれる。rustの名前階層はややこしいので注意が必要だ。

```rust
use gpio::{GPIOA, PIN_5, Level};
```
gpio.rs モジュールから指定したシンボルを使う宣言。Pythonと似た感じである。

```rust
#[no_mangle]
pub extern fn rust_main() {
```
CubeMX側の main()から、必要な初期化を終えたあとで rust_main を呼ぶようにすると、rust 側ではここから始められる。Cから呼ばれるので `#[no_mangle] pub extern`属性にする必要がある。

```rust
        GPIOA().WritePin(PIN_5, Level::High);
```
gpio.rs で定義したAPIを呼ぶ。

### src/cubemx/Src/main.c

上述したように、CubeMXの`main()`から `rust_main` を呼ぶ必要がある。次のようになっているはずなので、`/* USER CODE BEGIN 2*/`のあとに `rust_main()`の呼び出しを追加する。`/* USER CODE BEGIN */`から`/* USER CODE END */`の間に書いたコードは、CubeMXでコードを再生成しても維持される。

```c
int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();

  /* USER CODE BEGIN 2 */
  rust_main();
  /* USER CODE END 2 */

  while (1)
  {
  }
}
```

### build.rs

ビルド手順は次のようになる。

1. アセンブラとCのコードをgccでコンパイルする。
2. libcube.a にライブラリとしてまとめる。
3. (rust側をビルドしたあとに)ライブラリをリンクする。

それを build.rs に記述する。build.rsは rustで記述できるので、変数や繰り返しを用いて効率的に書けるのが良い。以下では、使うCのファイルをダラダラと列挙したが、build.rsが自力で探してくるようにするのも良いだろう。

```rust
use std::process::Command;
use std::env;
use std::path::Path;

fn main() {
    let cube_top = "src/cubemx";

    let out_dir = env::var("OUT_DIR").unwrap();

    let inc_dirs = [
        &format!("-I{}/Drivers/CMSIS/Device/ST/STM32F1xx/Include", cube_top),
        &format!("-I{}/Drivers/CMSIS/Include", cube_top),
        &format!("-I{}/Drivers/STM32F1xx_HAL_Driver/Inc", cube_top),
        &format!("-I{}/Inc", cube_top),
    ];

    let defines = [
        "-DSTM32F103xB"
    ];

    let srcs = [
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_cortex.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_dma.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_flash_ex.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_flash.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_gpio_ex.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_gpio.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_pwr.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_rcc_ex.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_rcc.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_tim_ex.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal_tim.c"],
        [&format!("{}/Drivers/STM32F1xx_HAL_Driver/Src", cube_top), "stm32f1xx_hal.c"],
        [&format!("{}/Src", cube_top), "stm32f1xx_hal_msp.c"],
        [&format!("{}/Src", cube_top), "stm32f1xx_it.c"],
        [&format!("{}/Src", cube_top), "system_stm32f1xx.c"],
        [&format!("{}/Src", cube_top), "main.c"],
    ];

    let mut objs: Vec<String> = Vec::new();

    Command::new("arm-none-eabi-as")
        .args(&["-mcpu=cortex-m3", "-mthumb", "-mfloat-abi=soft"])
        .args(&["src/cubemx/startup/startup_stm32f103xb.s"])
        .args(&["-o"])
        .arg(&format!("{}/startup_stm32f103xb.o", out_dir))
        .status().unwrap();

    for src in &srcs {
        let obj = src[1].to_string().replace(".c", ".o");

        Command::new("arm-none-eabi-gcc")
            .arg("-c")
            .args(&["-mcpu=cortex-m3", "-mthumb", "-mfloat-abi=soft"])
            .args(&defines)
            .args(&inc_dirs)
            .arg(&format!("{}/{}",src[0], src[1]))
            .arg("-o")
            .arg(&format!("{}/{}", out_dir, obj))
            .status().unwrap();

        objs.push(obj);
    }

    Command::new("arm-none-eabi-ar")
        .args(&["crus", "libcube.a"])
        .arg(&format!("{}/startup_stm32f103xb.o", out_dir))
        .args(&objs)
        .current_dir(&Path::new(&out_dir))
        .status().unwrap();

    println!("cargo:rustc-link-search=native={}", out_dir);
    println!("cargo:rustc-link-lib=static=cube");

    println!("cargo:rerun-if-changed=build.rs");
}
```
`xargo build`すればビルドが走り、`target/thumbv7m-none-eabi/debug/cubemx`をやきこめば、やっぱりLチカする。


## gdb

プログラム開発では、コーディングだけでなく、デバッグも避けて通れない。商用環境ではIDEを用いたデバッガが一般だが、オープンソース組み込み界隈では OpenOCDとgdbを用いたリモートデバッグが一般的だ。場合によってはGUIとしてEclipseを用いることもあると思うが、個人的にはEclipseはあまり好きではない。

OpenOCDとgdbを使ったデバッグの方法はCの場合と同じでrust特有のことは無いが簡単に解説する。

まず、別の端末でOpenOCDのサーバを起動する。
```
$ sudo openocd -f board/st_nucleo_f103rb.cfg
```
メインの端末で、(デバッグシンボルを含んでいる)オブジェクトファイルを引数に gdb を起動し、`target` を `remote localhost:3333`とする。OpenOCDがボードとつながっていて、gdb は OpenOCDのポート(localhost:3333)にコマンドを送ることでターゲットを操作するのだ。実行例に示すとおり、rustのソースコードレベルでデバッグができる。
```
$ <strong>arm-none-eabi-gdb target/thumbv7m-none-eabi/debug/cubemx</strong> 
GNU gdb (7.10-1ubuntu3+9) 7.10
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "--host=x86_64-linux-gnu --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from target/thumbv7m-none-eabi/debug/cubemx...done.
(gdb) <strong>target remote localhost:3333</strong>
Remote debugging using localhost:3333
0x00000000 in ?? ()
(gdb) continue
Continuing.
WARNING! The target is already running. All changes GDB did to registers will be discarded! Waiting for target to halt.
l
^C
Program received signal SIGINT, Interrupt.
cubemx::rust_main () at src/main.rs:20
20	        for _ in 1..4000000 {
(gdb) l
15	                asm!("");
16	            }
17	        }
18	
19	        GPIOA().WritePin(PIN_5, Level::Low);
20	        for _ in 1..4000000 {
21	            unsafe {
22	                asm!("");
23	            }
24	        }
(gdb) 
```
よく使うコマンドは次のとおり。

            <table border>
                <tr><td>cont(inue)</td><td>実行を継続</td></tr>
                <tr><td>^C</td><td>実行をブレイク</td></tr>
                <tr><td>b(reak) <i>func</i></td><td><i>func</i>にブレークポイントを設定する。</td></tr>
                <tr><td>n(ext) <i>d</i></td><td><i>d</i>行ステップオーバー。引数が無ければ1行</td></tr>
                <tr><td>s(tep)</td><td>ステップイン。</td></tr>
                <tr><td>n(ext)i</td><td>命令レベルで</td></tr>
                <tr><td>finish</td><td>現在のスタックフレームから脱出する。</td></tr>
                <tr><td>p(rint) <i>variable</i></td><td><i>variable</i>を表示する</td></tr>
                <tr><td>quit</td><td>gdbを終了する</td></tr>
            </table>

### gdb-dashboard

gdbをより使いやすくするのに gdb-dashboard([https://github.com/cyrus-and/gdb-dashboard](https://github.com/cyrus-and/gdb-dashboard)) というのがある。書かれているように、.gdbinit に設定を保存しておいて、soucece .gdbinit で読み込むことによって、多彩な情報をカラフルに表示するのだ。シンタックスハイライトで #???が見にくい時は、.gdbinitで、syntax_hiliging が 'vim'になっているところを''にすれば良い。

<div align="center"><img src="/images/2017-07-30-gdb-dashboard.png" alt="cubemx"></div>

### VS Code + Native Debug Plugin

もし VS Code を使っているのなら Native Debug Plugin という拡張を使うことで、VS Code 内蔵デバッガからgdbを使える。

<div align="center"><img src="/images/2017-07-30-vscode-debug.png" alt="cubemx"></div>

## サポートライブラリ

Lチカができたらプログラムが書けるかといえばそうではない。HALでも、最低限、UART, TIM は欲しいし、ADCも優先度が高い。CubeMXで生成されるのでプロジェクトごとに代わる部分はブロジェクトに属させ、そうでない部分は、ライブラリとして独立させるべきだ。また、HALだけでなく、Lock や Queue(FIFO)などのライブラリも欲しくなってくる。なぜならOS無し環境では `#![no_std]`なので、便利なユーティリティが供給されておらず、自作しなければならないからだ。

わたしのGitHubのリポジトリ([https://github.com/nkon/stm32cubef1](https://github.com/nkon/stm32cubef1))でも少しずつ整備してゆきたい。
