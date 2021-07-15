# Текстовый буфер

Теперь мы создадим пару структур: одну для представления символов, другая будет представлять текстовый буфер:

```rust
// d src/vga_buffer.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(C)]
struct ScreenChar {
    ascii_character: u8,
    color_code: ColorCode,
}

const BUFFER_HEIGHT: usize = 25;
const BUFFER_WIDTH: usize = 80;

#[repr(transparent)]
struct Buffer {
    chars: [[ScreenChar; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```

Rust не гарантирует сохранения порядка полей структуры в памяти, поэтому мы используем атрибут [`repr(C)`]. Он гарантирует, что поля структуры будут размещаться в памяти подобно Си-структуре, что гарантирует правильный порядок полей. Для структуры `Buffer` мы снова используем атрибут [`repr(transparent)`], чтобы убедиться, что размещение в памяти будет такое же, как у обычного массива.

[`repr(C)`]: https://doc.rust-lang.org/nightly/nomicon/other-reprs.html#reprc

Чтобы записывать что-то на экран, мы создадим отдельный тип писателя:

```rust
// в src/vga_buffer.rs

pub struct Writer {
    column_position: usize,
    color_code: ColorCode,
    buffer: &'static mut Buffer,
}
```

Писатель всегда будет использовать самую нижнюю строку. Он будет сдвигать строки вверх, если в строке не осталось мест (или при переводе строки). Поле `column_position` отслеживает текущую позицию в последней строке. Текущие основной и фоновый цвета хранятся в поле `color_code`, а ссылка на VGA-буфер сохраняется в поле `buffer`. Обратите внимание, что для поля `buffer` мы явно указываем время жизни, чтобы сообщить компилятору, как долго действительная ссылка. Кроме того, мы установили статическое время жизни, которое гарантирует, что ссылка действительна на протяжении всей работы программы.

[explicit lifetime]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#lifetime-annotation-syntax
[`'static`]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#the-static-lifetime
