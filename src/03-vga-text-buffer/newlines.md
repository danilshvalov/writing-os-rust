# Перевод строки

Прямо сейчас мы просто игнорируем перевод строки, как и все поместившиеся символы. Вместо этого мы хотим переместить каждый символ на одну строку вверх (самая верхняя строка будет просто удаляться), чтобы освободить текущую строку от символов. Давайте реализуем `new_line` метод у писателя:

```rust
// в src/vga_buffer.rs

impl Writer {
    fn new_line(&mut self) {
        for row in 1..BUFFER_HEIGHT {
            for col in 0..BUFFER_WIDTH {
                let character = self.buffer.chars[row][col].read();
                self.buffer.chars[row - 1][col].write(character);
            }
        }
        self.clear_row(BUFFER_HEIGHT - 1);
        self.column_position = 0;
    }

    fn clear_row(&mut self, row: usize) {/* TODO */}
}
```

We iterate over all screen characters and move each character one row up. Note that the range notation (`..`) is exclusive the upper bound. We also omit the 0th row (the first range starts at `1`) because it's the row that is shifted off screen.

Мы перебираем все символы на экране и перемещаем каждый на один ряд вверх. Обратите внимание, что конструкция `..` не включает верхнюю границу диапазона. Мы также исключаем из диапазона 0-ю строку, потому что она смещена за пределы экрана.

Теперь давайте реализуем метод `clear_row`:

```rust
// в src/vga_buffer.rs

impl Writer {
    fn clear_row(&mut self, row: usize) {
        let blank = ScreenChar {
            ascii_character: b' ',
            color_code: self.color_code,
        };
        for col in 0..BUFFER_WIDTH {
            self.buffer.chars[row][col].write(blank);
        }
    }
}
```

Этот метод очищает строку, перезаписывая все ее символы пробелом.
