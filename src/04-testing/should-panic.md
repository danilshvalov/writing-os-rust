# Тесты, которые должны падать

The test framework of the standard library supports a [`#[should_panic]` attribute][should_panic] that allows to construct tests that should fail. This is useful for example to verify that a function fails when an invalid argument is passed. Unfortunately this attribute isn't supported in `#[no_std]` crates since it requires support from the standard library.

[should_panic]: https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html#testing-panics

While we can't use the `#[should_panic]` attribute in our kernel, we can get similar behavior by creating an integration test that exits with a success error code from the panic handler. Let's start creating such a test with the name `should_panic`:

```rust
// in tests/should_panic.rs

#![no_std]
#![no_main]

use core::panic::PanicInfo;
use blog_os::{QemuExitCode, exit_qemu, serial_println};

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    serial_println!("[ok]");
    exit_qemu(QemuExitCode::Success);
    loop {}
}
```

This test is still incomplete as it doesn't define a `_start` function or any of the custom test runner attributes yet. Let's add the missing parts:

```rust
// in tests/should_panic.rs

#![feature(custom_test_frameworks)]
#![test_runner(test_runner)]
#![reexport_test_harness_main = "test_main"]

#[no_mangle]
pub extern "C" fn _start() -> ! {
    test_main();
    
    loop {}
}

pub fn test_runner(tests: &[&dyn Fn()]) {
    serial_println!("Running {} tests", tests.len());
    for test in tests {
        test();
        serial_println!("[test did not panic]");
        exit_qemu(QemuExitCode::Failed);
    }
    exit_qemu(QemuExitCode::Success);
}
```

Instead of reusing the `test_runner` from our `lib.rs`, the test defines its own `test_runner` function that exits with a failure exit code when a test returns without panicking (we want our tests to panic). If no test function is defined, the runner exits with a success error code. Since the runner always exits after running a single test, it does not make sense to define more than one `#[test_case]` function.

Now we can create a test that should fail:

```rust
// in tests/should_panic.rs

use blog_os::serial_print;

#[test_case]
fn should_fail() {
    serial_print!("should_panic::should_fail...\t");
    assert_eq!(0, 1);
}
```

The test uses the `assert_eq` to assert that `0` and `1` are equal. This of course fails, so that our test panics as desired. Note that we need to manually print the function name using `serial_print!` here because we don't use the `Testable` trait.

When we run the test through `cargo test --test should_panic` we see that it is successful because the test panicked as expected. When we comment out the assertion and run the test again, we see that it indeed fails with the _"test did not panic"_ message.

A significant drawback of this approach is that it only works for a single test function. With multiple `#[test_case]` functions, only the first function is executed because the execution cannot continue after the panic handler has been called. I currently don't know of a good way to solve this problem, so let me know if you have an idea!