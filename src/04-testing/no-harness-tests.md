# No Harness Tests == Автоматизированное тестирование

For integration tests that only have a single test function (like our `should_panic` test), the test runner isn't really needed. For cases like this, we can disable the test runner completely and run our test directly in the `_start` function.

The key to this is disable the `harness` flag for the test in the `Cargo.toml`, which defines whether a test runner is used for an integration test. When it's set to `false`, both the default test runner and the custom test runner feature are disabled, so that the test is treated like a normal executable.

Let's disable the `harness` flag for our `should_panic` test:

```toml
# in Cargo.toml

[[test]]
name = "should_panic"
harness = false
```

Now we vastly simplify our `should_panic` test by removing the test runner related code. The result looks like this:

```rust
// в tests/should_panic.rs

#![no_std]
#![no_main]

use core::panic::PanicInfo;
use blog_os::{exit_qemu, serial_print, serial_println, QemuExitCode};

#[no_mangle]
pub extern "C" fn _start() -> ! {
    should_fail();
    serial_println!("[test did not panic]");
    exit_qemu(QemuExitCode::Failed);
    loop{}
}

fn should_fail() {
    serial_print!("should_panic::should_fail...\t");
    assert_eq!(0, 1);
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    serial_println!("[ok]");
    exit_qemu(QemuExitCode::Success);
    loop {}
}
```

We now call the `should_fail` function directly from our `_start` function and exit with a failure exit code if it returns. When we run `cargo test --test should_panic` now, we see that the test behaves exactly as before.

Apart from creating `should_panic` tests, disabling the `harness` attribute can also be useful for complex integration tests, for example when the individual test functions have side effects and need to be run in a specified order.


