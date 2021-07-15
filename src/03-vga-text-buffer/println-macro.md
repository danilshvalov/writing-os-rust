# Макрос `println`

Теперь, когда у нас есть общедоступный писатель, давайте добавим поддержку макроса `println`. Мы не будем пытаться создать свой `println` с нуля, вместо этого посмотрим на реализацию макроса в стандартной библиотеке:

[macro syntax]: https://doc.rust-lang.org/nightly/book/ch19-06-macros.html#declarative-macros-with-macro_rules-for-general-metaprogramming
[`println!` macro]: https://doc.rust-lang.org/nightly/std/macro.println!.html

```rust
#[macro_export]
macro_rules! println {
    () => (print!("\n"));
    ($($arg:tt)*) => (print!("{}\n", format_args!($($arg)*)));
}
```

В макросах можно определить одно или несколько правил. Макрос `println` определяет два правила: первое правило для вызова без аргументов, то есть просто `println!()`. Этот вызов развернется в `print!("\n")` и просто распечатает символ перевода строки. Второе же правило используется при вызове с аргументами, например `println!("Hello")` или `println!("Number: {}", 4)`. Они также разворачиваются в вызов макроса `print!`, которому передаются все отформатированные аргументы вместе с символом перевода строки.

Атрибут `#[macro_export]` делает макрос доступным как для своего крейта, так и для внешних крейтов, а не только для модуля, в котором он определен. Он также помещает макрос в корневой модуль крейта, что позволяет использовать `use std::println` вместо `std::macros::println`.

Макрос `print!` определен так:

[`print!` macro]: https://doc.rust-lang.org/nightly/std/macro.print!.html

```rust
#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::io::_print(format_args!($($arg)*)));
}
```

Макрос разворачивается в вызов [функции `_print`][print-func], которая определена в модуле `io`. [Переменная `$crate`][crate-var] гарантирует, что макрос будет работать как внутри, так и снаружи пространства имен `std`.

[Макрос `format_args`][fmt-macro] создает [`fmt::Arguments`][fmt-args] из переданных аргументов. Он, в свою очередь, передается функции `_print`, которая вызывает `print_to`. Такая сложность объясняется поддержкой различных `Stdout`. Нам ни к чему такая сложность, поскольку мы просто хотим печатать в VGA-буфер.

[print-func]: https://github.com/rust-lang/rust/blob/29f5c699b11a6a148f097f82eaa05202f8799bbc/src/libstd/io/stdio.rs#L698
[crate-var]: https://doc.rust-lang.org/1.30.0/book/first-edition/macros.html#the-variable-crate
[fmt-marco]: https://doc.rust-lang.org/nightly/std/macro.format_args.html
[fmt-args]: https://doc.rust-lang.org/nightly/core/fmt/struct.Arguments.html

Чтобы печатать в наш буфер, мы просто скопируем реализации `println!` и `print!` макросов и немножко изменим их, добавив использование своей функции `_print`:

```rust
// in src/vga_buffer.rs

#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::vga_buffer::_print(format_args!($($arg)*)));
}

#[macro_export]
macro_rules! println {
    () => ($crate::print!("\n"));
    ($($arg:tt)*) => ($crate::print!("{}\n", format_args!($($arg)*)));
}

#[doc(hidden)]
pub fn _print(args: fmt::Arguments) {
    use core::fmt::Write;
    WRITER.lock().write_fmt(args).unwrap();
}
```

Одна вещь, которую мы изменили по сравнению с исходным определением `println`, заключается в том, что мы также добавили префикс `$crate` в вызове макроса `print!`. Теперь нам не нужно импортировать макрос `print!`, чтобы воспользоваться `println`.

Как и в стандартной библиотеке, мы добавили атрибут `#[macro_export]` для обоих макросов, чтобы сделать их доступными повсюду в нашем крейте. Заметим, что для импорта макросов теперь мы должны использовать `use crate::println` вместо `use crate::vga_buffer::println`, потому что, как говорилось ранее, макрос перемещается в корневой модуль крейта.

The `_print` function locks our static `WRITER` and calls the `write_fmt` method on it. This method is from the `Write` trait, we need to import that trait. The additional `unwrap()` at the end panics if printing isn't successful. But since we always return `Ok` in `write_str`, that should not happen.

Функция `_print` блокирует нашего `WRITER` и вызывает `write_fmt` метод у него. Этот метод берется из трейта `Write`, поэтому нам нужно импортировать его. В свою очередь, `unwrap()` вызывает панику, если печать не удалась. Но поскольку мы всегда возвращаем `Ok` в методе `write_str`, этого никогда не произойдет.

Так как мы должны иметь возможность вызвать `_print` вне нашего модуля, функцию необходимо сделать публичной. Однако, поскольку это является лишь деталью реализации, мы добавим [атрибут `doc(hidden)`][hidden-attr], чтобы скрыть его из генерируемой документации.

[hidden-attr]: https://doc.rust-lang.org/nightly/rustdoc/the-doc-attribute.html#dochidden

## Hello World с помощью `println`

Теперь мы можем использовать `println`  в нашей функции `_start`:

```rust
// в src/main.rs

#[no_mangle]
pub extern "C" fn _start() {
    println!("Hello World{}", "!");
    
    loop {}
}
```

Как и ожидалось, теперь мы видим надпись _«Hello World!»_ на экране:

![QEMU printing “Hello World!”](vga-hello-world.png)

## Выводим сообщения об ошибках на экран

С макросом `println` мы можем обновить нашу функцию, вызываемую при панике:

```rust
// в main.rs

/// This function is called on panic.
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{}", info);
    loop {}
}
```

Давайте протестируем нашу обновленную функцию, добавив  `panic!("Some panic message");` в функцию `_start`. Вы должны получить такое сообщение:

TODO добавить картинку
![QEMU printing panicked at 'Some panic message', src/main.rs:28:5](vga-panic.png)

Теперь при панике мы сможем увидеть сообщение об ошибке и местоположение этой ошибки.
