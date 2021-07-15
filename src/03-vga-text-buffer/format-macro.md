# Макрос форматирования

Было бы неплохо реализовать поддержку макросов форматирования Rust. Благодаря этому мы сможем выводить на экран разные типы, такие как целые числа или числа с плавающей запятой. Чтобы сделать это, нам нужно реализовать трейт [`core::fmt::Write`]. Единственный требуемый трейтом метод — это `write_str`, который очень похож на наш метод `write_string`, только в отличии от нашего возвращает `fmt::Result`:

[`core::fmt::Write`]: https://doc.rust-lang.org/nightly/core/fmt/trait.Write.html

```rust
// в src/vga_buffer.rs

use core::fmt;

impl fmt::Write for Writer {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        self.write_string(s);
        Ok(())
    }
}
```

Теперь мы можем использовать встроенные в Rust макросы форматирования `write!`/`writeln!`:

```rust
// в src/vga_buffer.rs

pub fn print_something() {
    use core::fmt::Write;
    let mut writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };
    
    writer.write_byte(b'H');
    writer.write_string("ello! ");
    write!(writer, "The numbers are {} and {}", 42, 1.0/3.0).unwrap();
}
```

Теперь на экран должно отображаться `Hello! The numbers are 42 and 0.3333333333333333`. Вызов `write!` возвращает `Result`, который сообщает предупреждение, если не используется, поэтому мы вызываем функцию [`unwrap`], которая паникует при ошибке. В нашем случае это не проблема, поскольку запись в VGA-буфер никогда не заканчивается неудачей.

[`unwrap`]: https://doc.rust-lang.org/core/result/enum.Result.html#method.unwrap
