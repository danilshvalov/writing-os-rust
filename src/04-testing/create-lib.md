# Перемещаем в библиотеку

To make the required functions available to our integration test, we need to split off a library from our `main.rs`, which can be included by other crates and integration test executables. To do this, we create a new `src/lib.rs` file:

```rust
// src/lib.rs

#![no_std]

```

Like the `main.rs`, the `lib.rs` is a special file that is automatically recognized by cargo. The library is a separate compilation unit, so we need to specify the `#![no_std]` attribute again.

To make our library work with `cargo test`, we need to also move the test functions and attributes from `main.rs`  to `lib.rs`:

```rust
// в src/lib.rs

#![cfg_attr(test, no_main)]
#![feature(custom_test_frameworks)]
#![test_runner(crate::test_runner)]
#![reexport_test_harness_main = "test_main"]

use core::panic::PanicInfo;

pub trait Testable {
    fn run(&self) -> ();
}

impl<T> Testable for T
where
    T: Fn(),
{
    fn run(&self) {
        serial_print!("{}...\t", core::any::type_name::<T>());
        self();
        serial_println!("[ok]");
    }
}

pub fn test_runner(tests: &[&dyn Testable]) {
    serial_println!("Running {} tests", tests.len());
    for test in tests {
        test.run();
    }
    exit_qemu(QemuExitCode::Success);
}

pub fn test_panic_handler(info: &PanicInfo) -> ! {
    serial_println!("[failed]\n");
    serial_println!("Error: {}\n", info);
    exit_qemu(QemuExitCode::Failed);
    loop {}
}

/// Точка входа для `cargo test`
#[cfg(test)]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    test_main();
    loop {}
}

#[cfg(test)]
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    test_panic_handler(info)
}
```

To make our `test_runner` available to executables and integration tests, we don't apply the `cfg(test)` attribute to it and make it public. We also factor out the implementation of our panic handler into a public `test_panic_handler` function, so that it is available for executables too.

Since our `lib.rs` is tested independently of our `main.rs`, we need to add a `_start` entry point and a panic handler when the library is compiled in test mode. By using the [`cfg_attr`] crate attribute, we conditionally enable the `no_main` attribute in this case.

[`cfg_attr`]: https://doc.rust-lang.org/reference/conditional-compilation.html#the-cfg_attr-attribute

We also move over the `QemuExitCode` enum and the `exit_qemu` function and make them public:

```rust
// в src/lib.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u32)]
pub enum QemuExitCode {
    Success = 0x10,
    Failed = 0x11,
}

pub fn exit_qemu(exit_code: QemuExitCode) {
    use x86_64::instructions::port::Port;

    unsafe {
        let mut port = Port::new(0xf4);
        port.write(exit_code as u32);
    }
}
```

Now executables and integration tests can import these functions from the library and don't need to define their own implementations. To also make `println` and `serial_println` available, we move the module declarations too:

```rust
// в src/lib.rs

pub mod serial;
pub mod vga_buffer;
```

We make the modules public to make them usable from outside of our library. This is also required for making our `println` and `serial_println` macros usable, since they use the `_print` functions of the modules.

Now we can update our `main.rs` to use the library:

```rust
// в src/main.rs

#![no_std]
#![no_main]
#![feature(custom_test_frameworks)]
#![test_runner(blog_os::test_runner)]
#![reexport_test_harness_main = "test_main"]

use core::panic::PanicInfo;
use blog_os::println;

#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!");

    #[cfg(test)]
    test_main();

    loop {}
}

/// Эта функция вызывается при панике
#[cfg(not(test))]
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{}", info);
    loop {}
}

#[cfg(test)]
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    blog_os::test_panic_handler(info)
}
```

The library is usable like a normal external crate. It is called like our crate, which is `blog_os` in our case. The above code uses the `blog_os::test_runner` function in the `test_runner` attribute and the `blog_os::test_panic_handler` function in our `cfg(test)` panic handler. It also imports the `println` macro to make it available to our `_start` and `panic` functions.

At this point, `cargo run` and `cargo test` should work again. Of course, `cargo test` still loops endlessly (you can exit with `ctrl+c`). Let's fix this by using the required library functions in our integration test.
