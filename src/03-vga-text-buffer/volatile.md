# Нежелательные оптимизации и volatile

Мы только что видели, что наше сообщение напечаталось правильно. Однако это код может сломаться, если в будущих версиях компилятора Rust оптимизатор будет действовать более агрессивно.

Дело в том, что мы используем `Buffer` только для записи и никогда не читаем из него. Компилятор не знает, что мы действительно обращаемся к VGA-буферу вместо обычной записи в ОЗУ. Поэтому он может решить, что эти записи не нужны и их можно опустить. Чтобы избежать этой ошибочной оптимизации, нам нужно пометить эти записи как _[volatile]_. Так мы сообщим компилятору, что запись имеет побочные эффекты, поэтому ее не нужно оптимизировать.

[volatile]: https://en.wikipedia.org/wiki/Volatile_(computer_programming)

Для этих целей мы используем крейт [volatile][volatile crate]. Этот крейт предоставляет обертку `Volatile` с методами `read` и `write`. Внутри этих методов используется функции [read_volatile] и [write_volatile] из `core` библиотеки Rust, которые гарантируют, что чтение и запись не будут оптимизированы.

[volatile crate]: https://docs.rs/volatile
[read_volatile]: https://doc.rust-lang.org/nightly/core/ptr/fn.read_volatile.html
[write_volatile]: https://doc.rust-lang.org/nightly/core/ptr/fn.write_volatile.html

Добавим в зависимости нашего проекта крейт `volatile`:

```toml
# в Cargo.toml

[dependencies]
volatile = "0.2.6"
```

Make sure to specify `volatile` version `0.2.6`. Newer versions of the crate are not compatible with this post.
The `0.2.6` is the [semantic] version number. For more information, see the [Specifying Dependencies] guide of the cargo documentation.

[semantic]: https://semver.org/
[Specifying Dependencies]: https://doc.crates.io/specifying-dependencies.html

Теперь нам нужно изменить наш тип `Buffer`, добавив volatile-обертку:

```rust
// в src/vga_buffer.rs

use volatile::Volatile;

struct Buffer {
    chars: [[Volatile<ScreenChar>; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```

Из-за обертки мы не можем пользоваться обычным `=`. Вместо этого нам нужно использовать метод `write`.

[generic]: https://doc.rust-lang.org/book/ch10-01-syntax.html

Значит нам нужно обновить наш метод `Writer::write_byte`:

```rust
// в src/vga_buffer.rs

impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        match byte {
            b'\n' => self.new_line(),
            byte => {
                ...
                
                self.buffer.chars[row][col].write(ScreenChar {
                    ascii_character: byte,
                    color_code,
                });
                ...
            }
        }
    }
    ...
}
```

Теперь мы можем быть уверены, что компилятор никогда не оптимизирует этот участок кода.
