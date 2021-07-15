# Абстракция для буфера VGA

The [VGA text mode] is a simple way to print text to the screen. In this post, we create an interface that makes its usage safe and simple, by encapsulating all unsafety in a separate module. We also implement support for Rust's [formatting macros].

[VGA text mode]: https://en.wikipedia.org/wiki/VGA-compatible_text_mode
[formatting macros]: https://doc.rust-lang.org/std/fmt/#related-macros

## The VGA Text Buffer

To print a character to the screen in VGA text mode, one has to write it to the text buffer of the VGA hardware. The VGA text buffer is a two-dimensional array with typically 25 rows and 80 columns, which is directly rendered to the screen. Each array entry describes a single screen character through the following format:

| Бит(ы) | Значение      |
| ------ | ------------- |
| 0-7    | ASCII код     |
| 8-11   | Основной цвет |
| 12-14  | Фоновый цвет  |
| 15     | Мерцание      |

The first byte represents the character that should be printed in the [ASCII encoding]. To be exact, it isn't exactly ASCII, but a character set named [_code page 437_] with some additional characters and slight modifications. For simplicity, we proceed to call it an ASCII character in this post.

[ASCII encoding]: https://en.wikipedia.org/wiki/ASCII
[_code page 437_]: https://en.wikipedia.org/wiki/Code_page_437

The second byte defines how the character is displayed. The first four bits define the foreground color, the next three bits the background color, and the last bit whether the character should blink. The following colors are available:

| Номер | Цвет         | Номер + бит яркости | Яркий цвет       |
| ----- | ------------ | ------------------- | ---------------- |
| 0x0   | Черный       | 0x8                 | Темно-серый      |
| 0x1   | Синий        | 0x9                 | Голубой          |
| 0x2   | Зеленый      | 0xa                 | Светло-зеленый   |
| 0x3   | Бирюзовый    | 0xb                 | Светло-бирюзовый |
| 0x4   | Красный      | 0xc                 | Светло-красный   |
| 0x5   | Пурпурный    | 0xd                 | Розовый          |
| 0x6   | Коричневый   | 0xe                 | Желтый           |
| 0x7   | Светло-серый | 0xf                 | Белый            |

Бит 4 — это бит яркости, который, например, превращает синий в голубой цвет. Для фонового цвета этот бит используется как бит для мерцания.

The VGA text buffer is accessible via [memory-mapped I/O] to the address `0xb8000`. This means that reads and writes to that address don't access the RAM, but directly the text buffer on the VGA hardware. This means that we can read and write it through normal memory operations to that address.

[memory-mapped I/O]: https://en.wikipedia.org/wiki/Memory-mapped_I/O

Note that memory-mapped hardware might not support all normal RAM operations. For example, a device could only support byte-wise reads and return junk when an `u64` is read. Fortunately, the text buffer [supports normal reads and writes], so that we don't have to treat it in special way.

[supports normal reads and writes]: https://web.stanford.edu/class/cs140/projects/pintos/specs/freevga/vga/vgamem.htm#manip

## A Rust Module

Now that we know how the VGA buffer works, we can create a Rust module to handle printing:

```rust
// in src/main.rs
mod vga_buffer;
```

For the content of this module we create a new `src/vga_buffer.rs` file. All of the code below goes into our new module (unless specified otherwise).


## Summary

In this post we learned about the structure of the VGA text buffer and how it can be written through the memory mapping at address `0xb8000`. We created a Rust module that encapsulates the unsafety of writing to this memory mapped buffer and presents a safe and convenient interface to the outside.

We also saw how easy it is to add dependencies on third-party libraries, thanks to cargo. The two dependencies that we added, `lazy_static` and `spin`, are very useful in OS development and we will use them in more places in future posts.

## What's next?

The next post explains how to set up Rust's built in unit test framework. We will then create some basic unit tests for the VGA buffer module from this post.
