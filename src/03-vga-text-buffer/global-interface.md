# Публичный интерфейс

Чтобы была возможность использовать писателя в других модулях без использования экземпляра `Writer`, мы создадим статическую переменную `WRITER`:

```rust
// в src/vga_buffer.rs

pub static WRITER: Writer = Writer {
    column_position: 0,
    color_code: ColorCode::new(Color::Yellow, Color::Black),
    buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
};
```

Однако теперь мы получим ошибку, если попробуем скомпилировать наш крейт:

```console
error[E0015]: calls in statics are limited to constant functions, tuple structs and tuple variants
 --> src/vga_buffer.rs:7:17
  |
7 |     color_code: ColorCode::new(Color::Yellow, Color::Black),
  |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error[E0396]: raw pointers cannot be dereferenced in statics
 --> src/vga_buffer.rs:8:22
  |
8 |     buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
  |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ dereference of raw pointer in constant

error[E0017]: references in statics may only refer to immutable values
 --> src/vga_buffer.rs:8:22
  |
8 |     buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
  |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ statics require immutable values

error[E0017]: references in statics may only refer to immutable values
 --> src/vga_buffer.rs:8:13
  |
8 |     buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
  |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ statics require immutable values
```

Чтобы понять, что здесь происходит, нам нужно знать как устроены статические переменные. Все статические переменные инициализируются во время компиляции, в отличии от обычных переменных, инициализация которых происходит во время выполнения.

[const evaluator]: https://rustc-dev-guide.rust-lang.org/const-eval.html
[Allow panicking in constants]: https://github.com/rust-lang/rfcs/pull/2345

Проблема с `ColorCode::new` может быть решена с помощью использования `const`-функции, но основная проблема в том, что компилятор Rust не может во время компиляции превратить сырые указатели в ссылки.

[`const` functions]: https://doc.rust-lang.org/unstable-book/language-features/const-fn.html

## Использование `lazy_static`

К счастью, для решения проблемы с статической инициализацией существует крейт [lazy_static]. Этот крейт предоставляет макрос `lazy_static!`, который предоставляет ленивую статическую инициализацию. Вместо вычисления во время компиляции, инициализация происходит во время первого обращения, то есть во время исполнения.

[lazy_static]: https://docs.rs/lazy_static/1.0.1/lazy_static/

Давайте добавим крейт `lazy_static` в наш проект:

```toml
# в Cargo.toml

[dependencies.lazy_static]
version = "1.0"
features = ["spin_no_std"]
```

Поскольку мы не подключаем стандартную библиотеку, нам нужна функция `spin_no_std`.

Теперь, имя `lazy_static`, мы можем без проблем создать `WRITER`:

```rust
// в src/vga_buffer.rs

use lazy_static::lazy_static;

lazy_static! {
    pub static ref WRITER: Writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };
}
```

Однако этот довольно бесполезен, если его нельзя изменять: мы не можем ничего записать в VGA-буфер, поскольку все методы, необходимые для этого, принимают `&mut self`. Возможное решение — [добавить `mut`][mutable static] к нашей переменной, однако после этого каждое чтение и каждая запись будет небезопасной, поскольку может возникнуть гонка за данными. Использование `static mut` крайне не рекомендуется. Существуют ли другие решения? Мы могли бы обернуть нашу переменную в [`RefCell`][RefCell] или в [`UnsafeCell`][UnsafeCell], которые обеспечивают внутреннюю изменчивость, но эти типы не реализуют трейт [`Sync`][Sync] (по уважительно причине), а значит мы не можем использовать их со статическими переменными.

[mutable static]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable
[remove static mut]: https://internals.rust-lang.org/t/pre-rfc-remove-static-mut/1437
[RefCell]: https://doc.rust-lang.org/book/ch15-05-interior-mutability.html#keeping-track-of-borrows-at-runtime-with-refcellt
[UnsafeCell]: https://doc.rust-lang.org/nightly/core/cell/struct.UnsafeCell.html
[interior mutability]: https://doc.rust-lang.org/book/ch15-05-interior-mutability.html
[Sync]: https://doc.rust-lang.org/nightly/core/marker/trait.Sync.html

## Спин-блокировка

Чтобы получить синхронизированную внутреннюю изменчивость, пользователи стандартной библиотеки могут использовать [`Mutex`][Mutex]. Он обеспечивает взаимное исключение, блокируя поток, если кто-то пользуется защищенными мьютексом ресурсами. Но в нашем ядре нет поддержки блокировок, нет даже концепции потоков, значит мы не можем использовать `Mutex`. Однако в информатике существует вид мьютексов, не требующий функции операционной системы: **[спин-блокировка][spinlock]**. Вместо блокировки, поток просто крутится в цикле до тех пор, пока занятые ресурсы не будут освобождены.

[Mutex]: https://doc.rust-lang.org/nightly/std/sync/struct.Mutex.html
[spinlock]: https://ru.wikipedia.org/wiki/Spinlock

Мы добавим крейт [spin][spin crate] в зависимости проекта, чтобы использовать спин-мьютекс:

[spin crate]: https://crates.io/crates/spin

```toml
# in Cargo.toml
[dependencies]
spin = "0.5.2"
```

Осталось только добавить мьютекс нашему писателю:

```rust
// в src/vga_buffer.rs

use spin::Mutex;
...
lazy_static! {
    pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    });
}
```

Теперь мы можем удалить функцию `print_something` и печатать прямо из нашей функции `_start`:

```rust
// в src/main.rs
#[no_mangle]
pub extern "C" fn _start() -> ! {
    use core::fmt::Write;
    vga_buffer::WRITER.lock().write_str("Hello again").unwrap();
    write!(vga_buffer::WRITER.lock(), ", some numbers: {} {}", 42, 1.337).unwrap();

    loop {}
}
```

Нам нужно импортировать трейт `fmt::Write`, чтобы иметь возможность использовать функции для записи.

## Безопасность

Обратите внимание, что у нас есть только один блок `unsafe`, который необходим для создания ссылки на участок памяти по адресу `0xb8000` в конструкторе `Buffer` . Все остальные операции являются безопасными. По умолчанию Rust использует проверку границ массива при доступе по индексу, поэтому мы не можем случайно изменить данные за пределами буфера. Таким образом, с помощью системы типов мы обеспечили безопасный интерфейс для нашего VGA-буфера.
