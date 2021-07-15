# Тестирование VGA-буфера

Now that we have a working test framework, we can create a few tests for our VGA buffer implementation. First, we create a very simple test to verify that `println` works without panicking:

```rust
// в src/vga_buffer.rs

#[test_case]
fn test_println_simple() {
    println!("test_println_simple output");
}
```

The test just prints something to the VGA buffer. If it finishes without panicking, it means that the `println` invocation did not panic either.

To ensure that no panic occurs even if many lines are printed and lines are shifted off the screen, we can create another test:

```rust
// в src/vga_buffer.rs

#[test_case]
fn test_println_many() {
    for _ in 0..200 {
        println!("test_println_many output");
    }
}
```

We can also create a test function to verify that the printed lines really appear on the screen:

```rust
// в src/vga_buffer.rs

#[test_case]
fn test_println_output() {
    let s = "Some test string that fits on a single line";
    println!("{}", s);
    for (i, c) in s.chars().enumerate() {
        let screen_char = WRITER.lock().buffer.chars[BUFFER_HEIGHT - 2][i].read();
        assert_eq!(char::from(screen_char.ascii_character), c);
    }
}
```

The function defines a test string, prints it using `println`, and then iterates over the screen characters of the static `WRITER`, which represents the vga text buffer. Since `println` prints to the last screen line and then immediately appends a newline, the string should appear on line `BUFFER_HEIGHT - 2`.

By using [`enumerate`], we count the number of iterations in the variable `i`, which we then use for loading the screen character corresponding to `c`. By comparing the `ascii_character` of the screen character with `c`, we ensure that each character of the string really appears in the vga text buffer.

[`enumerate`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.enumerate

As you can imagine, we could create many more test functions, for example a function that tests that no panic occurs when printing very long lines and that they're wrapped correctly. Or a function for testing that newlines, non-printable characters, and non-unicode characters are handled correctly.

For the rest of this post, however, we will explain how to create _integration tests_ to test the interaction of different components together.