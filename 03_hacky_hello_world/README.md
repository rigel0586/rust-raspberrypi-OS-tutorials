# Tutorial 03 - Hacky Hello World

## tl;dr

- Introducing global `print!()` macros to enable "printf debugging" at the earliest.
- To keep tutorial length reasonable, printing functions for now "abuse" a QEMU property that lets
  us use the RPi's `UART` without setting it up properly.
- Using  the real hardware `UART` is enabled step-by-step in following tutorials.

## Notable additions

- `src/console.rs` introduces interface `Traits` for console commands.
- `src/bsp/raspberrypi/console.rs` implements the interface for QEMU's emulated UART.
- The panic handler makes use of the new `print!()` to display user error messages.

## Test it

QEMU is no longer running in assembly mode. It will from now on show the output of the `console`.

```console
$ make qemu
[...]
Hello from Rust!

Kernel panic: Stopping here.
```

## Diff to previous
```diff

diff -uNr 02_runtime_init/Makefile 03_hacky_hello_world/Makefile
--- 02_runtime_init/Makefile
+++ 03_hacky_hello_world/Makefile
@@ -11,7 +11,7 @@
     KERNEL_BIN        = kernel8.img
     QEMU_BINARY       = qemu-system-aarch64
     QEMU_MACHINE_TYPE = raspi3
-    QEMU_RELEASE_ARGS = -d in_asm -display none
+    QEMU_RELEASE_ARGS = -serial stdio -display none
     OBJDUMP_BINARY    = aarch64-none-elf-objdump
     NM_BINARY         = aarch64-none-elf-nm
     LINKER_FILE       = src/bsp/raspberrypi/link.ld
@@ -21,7 +21,7 @@
     KERNEL_BIN        = kernel8.img
     QEMU_BINARY       = qemu-system-aarch64
     QEMU_MACHINE_TYPE =
-    QEMU_RELEASE_ARGS = -d in_asm -display none
+    QEMU_RELEASE_ARGS = -serial stdio -display none
     OBJDUMP_BINARY    = aarch64-none-elf-objdump
     NM_BINARY         = aarch64-none-elf-nm
     LINKER_FILE       = src/bsp/raspberrypi/link.ld

diff -uNr 02_runtime_init/src/bsp/raspberrypi/console.rs 03_hacky_hello_world/src/bsp/raspberrypi/console.rs
--- 02_runtime_init/src/bsp/raspberrypi/console.rs
+++ 03_hacky_hello_world/src/bsp/raspberrypi/console.rs
@@ -0,0 +1,47 @@
+// SPDX-License-Identifier: MIT OR Apache-2.0
+//
+// Copyright (c) 2018-2021 Andre Richter <andre.o.richter@gmail.com>
+
+//! BSP console facilities.
+
+use crate::console;
+use core::fmt;
+
+//--------------------------------------------------------------------------------------------------
+// Private Definitions
+//--------------------------------------------------------------------------------------------------
+
+/// A mystical, magical device for generating QEMU output out of the void.
+struct QEMUOutput;
+
+//--------------------------------------------------------------------------------------------------
+// Private Code
+//--------------------------------------------------------------------------------------------------
+
+/// Implementing `core::fmt::Write` enables usage of the `format_args!` macros, which in turn are
+/// used to implement the `kernel`'s `print!` and `println!` macros. By implementing `write_str()`,
+/// we get `write_fmt()` automatically.
+///
+/// See [`src/print.rs`].
+///
+/// [`src/print.rs`]: ../../print/index.html
+impl fmt::Write for QEMUOutput {
+    fn write_str(&mut self, s: &str) -> fmt::Result {
+        for c in s.chars() {
+            unsafe {
+                core::ptr::write_volatile(0x3F20_1000 as *mut u8, c as u8);
+            }
+        }
+
+        Ok(())
+    }
+}
+
+//--------------------------------------------------------------------------------------------------
+// Public Code
+//--------------------------------------------------------------------------------------------------
+
+/// Return a reference to the console.
+pub fn console() -> impl console::interface::Write {
+    QEMUOutput {}
+}

diff -uNr 02_runtime_init/src/bsp/raspberrypi.rs 03_hacky_hello_world/src/bsp/raspberrypi.rs
--- 02_runtime_init/src/bsp/raspberrypi.rs
+++ 03_hacky_hello_world/src/bsp/raspberrypi.rs
@@ -4,4 +4,5 @@

 //! Top-level BSP file for the Raspberry Pi 3 and 4.

+pub mod console;
 pub mod memory;

diff -uNr 02_runtime_init/src/console.rs 03_hacky_hello_world/src/console.rs
--- 02_runtime_init/src/console.rs
+++ 03_hacky_hello_world/src/console.rs
@@ -0,0 +1,19 @@
+// SPDX-License-Identifier: MIT OR Apache-2.0
+//
+// Copyright (c) 2018-2021 Andre Richter <andre.o.richter@gmail.com>
+
+//! System console.
+
+//--------------------------------------------------------------------------------------------------
+// Public Definitions
+//--------------------------------------------------------------------------------------------------
+
+/// Console interfaces.
+pub mod interface {
+    /// Console write functions.
+    ///
+    /// `core::fmt::Write` is exactly what we need for now. Re-export it here because
+    /// implementing `console::Write` gives a better hint to the reader about the
+    /// intention.
+    pub use core::fmt::Write;
+}

diff -uNr 02_runtime_init/src/main.rs 03_hacky_hello_world/src/main.rs
--- 02_runtime_init/src/main.rs
+++ 03_hacky_hello_world/src/main.rs
@@ -101,21 +101,25 @@
 //! # Boot flow
 //!
 //! 1. The kernel's entry point is the function [`cpu::boot::arch_boot::_start()`].
-//!     - It is implemented in `src/_arch/__arch_name__/cpu/boot.S`.
+//!     - It is implemented in `src/_arch/__arch_name__/cpu/boot.rs`.
 //! 2. Once finished with architectural setup, the arch code calls [`runtime_init::runtime_init()`].
 //!
-//! [`cpu::boot::arch_boot::_start()`]: ../src/kernel/cpu/up/_arch/aarch64/cpu/boot.rs.html
+//! [`cpu::boot::arch_boot::_start()`]: cpu/boot/arch_boot/fn._start.html
 //! [`runtime_init::runtime_init()`]: runtime_init/fn.runtime_init.html

 #![feature(asm)]
+#![feature(format_args_nl)]
 #![feature(global_asm)]
+#![feature(panic_info_message)]
 #![no_main]
 #![no_std]

 mod bsp;
+mod console;
 mod cpu;
 mod memory;
 mod panic_wait;
+mod print;
 mod runtime_init;

 /// Early init code.
@@ -124,5 +128,7 @@
 ///
 /// - Only a single core must be active and running this function.
 unsafe fn kernel_init() -> ! {
-    panic!()
+    println!("[0] Hello from Rust!");
+
+    panic!("Stopping here.")
 }

diff -uNr 02_runtime_init/src/panic_wait.rs 03_hacky_hello_world/src/panic_wait.rs
--- 02_runtime_init/src/panic_wait.rs
+++ 03_hacky_hello_world/src/panic_wait.rs
@@ -4,10 +4,16 @@

 //! A panic handler that infinitely waits.

-use crate::cpu;
+use crate::{cpu, println};
 use core::panic::PanicInfo;

 #[panic_handler]
-fn panic(_info: &PanicInfo) -> ! {
+fn panic(info: &PanicInfo) -> ! {
+    if let Some(args) = info.message() {
+        println!("\nKernel panic: {}", args);
+    } else {
+        println!("\nKernel panic!");
+    }
+
     cpu::wait_forever()
 }

diff -uNr 02_runtime_init/src/print.rs 03_hacky_hello_world/src/print.rs
--- 02_runtime_init/src/print.rs
+++ 03_hacky_hello_world/src/print.rs
@@ -0,0 +1,38 @@
+// SPDX-License-Identifier: MIT OR Apache-2.0
+//
+// Copyright (c) 2018-2021 Andre Richter <andre.o.richter@gmail.com>
+
+//! Printing.
+
+use crate::{bsp, console};
+use core::fmt;
+
+//--------------------------------------------------------------------------------------------------
+// Public Code
+//--------------------------------------------------------------------------------------------------
+
+#[doc(hidden)]
+pub fn _print(args: fmt::Arguments) {
+    use console::interface::Write;
+
+    bsp::console::console().write_fmt(args).unwrap();
+}
+
+/// Prints without a newline.
+///
+/// Carbon copy from <https://doc.rust-lang.org/src/std/macros.rs.html>
+#[macro_export]
+macro_rules! print {
+    ($($arg:tt)*) => ($crate::print::_print(format_args!($($arg)*)));
+}
+
+/// Prints with a newline.
+///
+/// Carbon copy from <https://doc.rust-lang.org/src/std/macros.rs.html>
+#[macro_export]
+macro_rules! println {
+    () => ($crate::print!("\n"));
+    ($($arg:tt)*) => ({
+        $crate::print::_print(format_args_nl!($($arg)*));
+    })
+}

```
