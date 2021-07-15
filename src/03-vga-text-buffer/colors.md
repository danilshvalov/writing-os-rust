# Цвета

Самый лучший способ представить наши виды доступных цветов — с помощью перечисления:

```rust
// в src/vga_buffer.rs

#[allow(dead_code)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum Color {
    Black = 0,
    Blue = 1,
    Green = 2,
    Cyan = 3,
    Red = 4,
    Magenta = 5,
    Brown = 6,
    LightGray = 7,
    DarkGray = 8,
    LightBlue = 9,
    LightGreen = 10,
    LightCyan = 11,
    LightRed = 12,
    Pink = 13,
    Yellow = 14,
    White = 15,
}
```

Мы использовали [перечисление в стиле Си][C-like enum], чтобы явно указать номер для каждого цвета. Благодаря атрибуту `repr(u8)` каждый цвет сохраняется как `u8`. На самом деле 4 бит нам было бы достаточно, но в Rust нет `u4`-типа.

[C-like enum]: https://doc.rust-lang.org/rust-by-example/custom_types/enum/c_like.html

Обычно компилятор выдает предупреждение для каждой неиспользуемой переменной, структуры и т.п. Используя атрибут `#[allow(dead_code)]`, мы отключаем эти предупреждения для `Color`.

By [deriving] the [`Copy`], [`Clone`], [`Debug`], [`PartialEq`], and [`Eq`] traits, we enable [copy semantics] for the type and make it printable and comparable.

[deriving]: https://doc.rust-lang.org/rust-by-example/trait/derive.html
[`Copy`]: https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html
[`Clone`]: https://doc.rust-lang.org/nightly/core/clone/trait.Clone.html
[`Debug`]: https://doc.rust-lang.org/nightly/core/fmt/trait.Debug.html
[`PartialEq`]: https://doc.rust-lang.org/nightly/core/cmp/trait.PartialEq.html
[`Eq`]: https://doc.rust-lang.org/nightly/core/cmp/trait.Eq.html
[copy semantics]: https://doc.rust-lang.org/1.30.0/book/first-edition/ownership.html#copy-types

Для представления кодировки цвета, которая включает в себя основной и фоновый цвет, мы воспользуемся идиомой [New Type][newtype] поверх `u8`:

```rust
// в src/vga_buffer.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(transparent)]
struct ColorCode(u8);

impl ColorCode {
    fn new(foreground: Color, background: Color) -> ColorCode {
        ColorCode((background as u8) << 4 | (foreground as u8))
    }
}
```

Структура `ColorCode` содержит информацию о основном и фоновом цвете. Как и раньше, мы используем атрибут `derive`, чтобы получить реализации `Copy` и `Debug` трейтов. Чтобы гарантировать, что структура имеет то же выравнивание, как у `u8`, мы используем атрибут [`repr(transparent)`].

[`repr(transparent)`]: https://doc.rust-lang.org/nomicon/other-reprs.html#reprtransparent
[newtype]: https://doc.rust-lang.ru/stable/rust-by-example/generics/new_types.html
