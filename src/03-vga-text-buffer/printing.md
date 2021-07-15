# Вывод на экран

Теперь мы можем использовать нашего писателя для изменения буфера символов. Сначала мы создадим метод для записи одного байта:

```rust
// в src/vga_buffer.rs

impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        match byte {
            b'\n' => self.new_line(),
            byte => {
                if self.column_position >= BUFFER_WIDTH {
                    self.new_line();
                }
                
                let row = BUFFER_HEIGHT - 1;
                let col = self.column_position;

                let color_code = self.color_code;
                self.buffer.chars[row][col] = ScreenChar {
                    ascii_character: byte,
                    color_code,
                };
                self.column_position += 1;
            }
        }
    }
    
    fn new_line(&mut self) {/* TODO */}
}
```

Если символ является символом перевод строки, т.е. `\n`, то писатель ничего не выводит на экран. Вместо этого он вызывает метод `new_line`, который мы реализуем позже. Остальные символы мы просто выводим на экран.

[newline]: https://en.wikipedia.org/wiki/Newline

Перед тем, как вывести очередной символ, писатель проверяет, что в строке еще есть место. В случае, если места больше нет, писатель вновь вызывает метод `newline` для перевода строки, после чего записывает новый символ `ScreenChar` в буфер, а затем увеличивает текущую позицию в строке.

Чтобы вывести строку на экран, мы можем представить ее в виде байтов, а затем вывести каждый символ отдельно:

```rust
// в src/vga_buffer.rs

impl Writer {
    pub fn write_string(&mut self, s: &str) {
        for byte in s.bytes() {
            match byte {
                // printable ASCII byte or newline
                0x20..=0x7e | b'\n' => self.write_byte(byte),
                // not part of printable ASCII range
                _ => self.write_byte(0xfe),
            }
        
        }
    }
}
```

VGA-буфер поддерживает только ASCII-символы и дополнительные байты [кодовой страницы 437][code page 437]. По умолчанию Rust хранит строки в кодировке [UTF-8], поэтому они могут содержать неподдерживаемые VGA-буфером символы. Мы используем конструкцию `match`, чтобы отличить ASCII-символы. Неподдерживаемые символы мы заменяем на символ `■`, который имеет шестнадцатеричный код `0xfe` на оборудовании VGA.

[code page 437]: https://en.wikipedia.org/wiki/Code_page_437
[UTF-8]: https://www.fileformat.info/info/unicode/utf8.htm

## Попробуем это

Чтобы протестировать вывод символов на экран, мы создадим временную функцию:

```rust
// в src/vga_buffer.rs

pub fn print_something() {
    let mut writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };
    
    writer.write_byte(b'H');
    writer.write_string("ello ");
    writer.write_string("Wörld!");
}
```

Сначала мы создаем писателя с указателем на VGA-буфер. Синтаксически это может выглядеть немного странно: сначала мы преобразуем число `0xb8000` в изменяемый [сырой указатель][raw pointer], затем разыменовываем его с помощью  `*` и сразу же заимствуем с помощью `&mut`. Мы обернули все в блок `unsafe`, потому что компилятор не может гарантировать, что разыменовывание сырого указателя не приведет к ошибке.

[raw pointer]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer
[`unsafe` block]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html

Затем мы записываем символ `b'H'`. Префикс `b` представляет строки в виде байтов, а не в виде обычных символов. Записывая строки `"ello "` и `"Wörld!"`, мы тестируем наш метод `write_string`, а также обработку неподдерживаемых символов. Чтобы увидеть эти символы на экране, нам нужно вызвать функцию `print_something` из нашей функции `_start`:

```rust
// в src/main.rs

#[no_mangle]
pub extern "C" fn _start() -> ! {
    vga_buffer::print_something();
    
    loop {}
}
```

When we run our project now, a `Hello W■■rld!` should be printed in the _lower_ left corner of the screen in yellow:

Запустив наше ядро, вы должны увидеть `Hello W■■rld!` в нижнем левом углу экрана. Текст будет желтого цвета, а фон — черного:

[byte literal]: https://doc.rust-lang.org/reference/tokens.html#byte-literals
TODO добавить изображение
![QEMU output with a yellow `Hello W■■rld!` in the lower left corner](vga-hello.png)

Обратите внимание, что символ `ö` отобразился как два символа `■`. Дело в том, что символ `ö` представлен двумя байтами в кодировке [UTF-8], оба из которых не соответствуют ни одному ASCII-символу. Фактически, это главная отличительная черта UTF-8: отдельные байты мультибайтовых символов никогда не соответствуют ASCII-символам.
