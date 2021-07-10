# Panic и eh_personality

Давайте исправим эти две ошибки! Начнем с простого — реализации паники, а затем узнаем, что за `eh_personality` от нас требует компилятор.

## Реализация паники

Атрибут `panic_handler` определяет функцию, которую компилятор должен вызвать при возникновении [паники][panic] входе работы программы. Стандартная библиотека предоставляет собственную функцию обработки паники, но так как мы используем `no_std`, нам нужно определить ее самостоятельно:

```rust
// в main.rs

use core::panic::PanicInfo;

/// Эта функция вызывается при панике
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

[Параметр `PanicInfo`][PanicInfo] содержит в себе информацию о файле и строку, где произошла паника, а также может содержать сообщение с ошибкой. Функция должна никогда не возвращать результат, поэтому она помечена как [расходящаяся функция][diverging function] с помощью возврата [пустого типа][never-type] `!`. Сейчас мы мало что можем сделать с этой функцией, поэтому просто создадим бесконечный цикл.

## Языковой элемент `eh_personality`

Языковые элементы - это специальные функции и типы, которые требует компилятор. Одним из таких является трейт [`Copy`], который сообщает компилятору какие типы имеют [_семантику копирования_][`Copy`]. Если посмотреть на [реализацию][copy impl], мы увидим, что трейт помечен специальным атрибутом `#[lang = "copy"]`, который и определяет его как языковой элемент.

Хотя создание пользовательских реализаций языковых элементов возможно, делать это следует только в крайнем случае. Причина в том, что языковые элементы представляют собой очень нестабильные детали реализации.

Языковой элемент [`eh_personality`][eh_personality] помечает функцию, которая используется для реализации [раскрутки стека][stack unwinding]. По умолчанию Rust использует раскрутку стека для запуска деструкторов всех переменных, живущих на стеке, в случае паники. Это гарантирует, что вся используемая память будет освобождена, позволяет родительскому потоку перехватить панику и продолжить выполнение. Однако раскрутка стека - это сложный процесс, требующий наличие специальных библиотек, специфичных для ОС (например [libunwind] на Linux или [structured exception handling] на Windows), так что мы не будем использовать раскрутку в нашей ОС.

## Отключение раскрутки стека

Бывают случаи, когда раскрутка нежелательна, поэтому Rust предоставляет возможность [прерывать работу][abort on panic] исполняемого файла при панике. Кроме того, это отключает генерацию символов, необходимых для раскрутки стека, и, таким образом, значительно уменьшает размер бинарного файла. Есть несколько способов отключения раскрутки. Самый простой - добавить несколько строк в наш `Cargo.toml`:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

 Мы указываем прекратить работу исполняемого файла при панике для `dev` профиля (используется при `cargo build`) и для `release` профиля (используется при `cargo build --release`). Теперь языковой элемент `eh_personality` больше не требуется.

Теперь мы исправили обе указанные выше ошибки. Однако, если мы попытаемся скомпилировать программу, возникнет другая ошибка:

```console
$ cargo build
error: requires `start` lang_item
```

Компилятор не может найти языковой элемент `start`, который определяет точку входа в программу.

[PanicInfo]: https://doc.rust-lang.org/nightly/core/panic/struct.PanicInfo.html
[panic]: https://doc.rust-lang.org/stable/book/ch09-01-unrecottps://doc.rust-lang.org/nightly/core/panic/struct.PanicInfo.html
[eh_personality]: https://github.com/rust-lang/rust/blob/edb368491551a77d77a48446d4ee88b35490c565/src/libpanic_unwind/gcc.rs#L11-L45
[stack unwinding]: https://www.bogotobogo.com/cplusplus/stackunwinding.php
[abort on panic]: https://github.com/rust-lang/rust/pull/32900
[`Copy`]: https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html
[copy impl]: https://github.com/rust-lang/rust/blob/485397e49a02a3b7ff77c17e4a3f16c653925cb3/src/libcore/marker.rs#L296-L299
[libunwind]: https://www.nongnu.org/libunwind/
[structured exception handling]: https://docs.microsoft.com/de-de/windows/win32/debug/structured-exception-handling
[diverging function]: https://doc.rust-lang.org/1.30.0/book/first-edition/functions.html#diverging-functions
[never-type]: https://doc.rust-lang.org/nightly/std/primitive.never.html
