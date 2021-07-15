# Доделываем интеграционные тесты

Like our `src/main.rs`, our `tests/basic_boot.rs` executable can import types from our new library. This allows us to import the missing components to complete our test:

```rust
// in tests/basic_boot.rs

#![test_runner(blog_os::test_runner)]

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    blog_os::test_panic_handler(info)
}
```

Instead of reimplementing the test runner, we use the `test_runner` function from our library by changing the `#![test_runner(crate::test_runner)]` attribute to `#![test_runner(blog_os::test_runner)]`. We then don't need the `test_runner` stub function in `basic_boot.rs` anymore, so we can remove it. For our `panic` handler, we call the `blog_os::test_panic_handler` function like we did in our `main.rs`.

Now `cargo test` exits normally again. When you run it, you see that it builds and runs the tests for our `lib.rs`, `main.rs`, and `basic_boot.rs` separately after each other. For the `main.rs` and the `basic_boot` integration test, it reports "Running 0 tests" since these files don't have any functions annotated with `#[test_case]`.

We can now add tests to our `basic_boot.rs`. For example, we can test that `println` works without panicking, like we did in the vga buffer tests:

```rust
// in tests/basic_boot.rs

use blog_os::println;

#[test_case]
fn test_println() {
    println!("test_println output");
}
```

When we run `cargo test` now, we see that it finds and executes the test function.

The test might seem a bit useless right now since it's almost identical to one of the VGA buffer tests. However, in the future the `_start` functions of our `main.rs` and `lib.rs` might grow and call various initialization routines before running the `test_main` function, so that the two tests are executed in very different environments.

By testing `println` in a `basic_boot` environment without calling any initialization routines in `_start`, we can ensure that `println` works right after booting. This is important because we rely on it e.g. for printing panic messages.

## Будущие тесты

The power of integration tests is that they're treated as completely separate executables. This gives them complete control over the environment, which makes it possible to test that the code interacts correctly with the CPU or hardware devices.

Our `basic_boot` test is a very simple example for an integration test. In the future, our kernel will become much more featureful and interact with the hardware in various ways. By adding integration tests, we can ensure that these interactions work (and keep working) as expected. Some ideas for possible future tests are:

- **CPU Exceptions**: When the code performs invalid operations (e.g. divides by zero), the CPU throws an exception. The kernel can register handler functions for such exceptions. An integration test could verify that the correct exception handler is called when a CPU exception occurs or that the execution continues correctly after resolvable exceptions.
- **Page Tables**: Page tables define which memory regions are valid and accessible. By modifying the page tables, it is possible to allocate new memory regions, for example when launching programs. An integration test could perform some modifications of the page tables in the `_start` function and then verify that the modifications have the desired effects in `#[test_case]` functions.
- **Userspace Programs**: Userspace programs are programs with limited access to the system's resources. For example, they don't have access to kernel data structures or to the memory of other programs. An integration test could launch userspace programs that perform forbidden operations and verify that the kernel prevents them all.

As you can imagine, many more tests are possible. By adding such tests, we can ensure that we don't break them accidentally when we add new features to our kernel or refactor our code. This is especially important when our kernel becomes larger and more complex.