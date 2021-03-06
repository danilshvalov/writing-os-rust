# Небольшое ядро

Теперь, когда мы примерно знаем, как загружается компьютер, пришло время создать наше собственное небольшое ядро. Наша цель - создать образ диска, который выводит "Hello World!" на экран при загрузке. Для этого мы воспользуемся нашим [автономным бинарным файлом] из предыдущей главы.

Как вы помните, мы создали автономный бинарный файл с помощью `cargo`, но в зависимости от ОС нам потребовались разные имена точек входа и разные флаги компилятора. Дело в том, что по умолчанию `cargo` создает исполняемый файл для _хост-системы_, т. е. для системы, в которой вы сейчас работаете. Это не то, что мы хотим, потому что ядро на начальных стадиях загрузки компьютера, где еще нет никакой ОС. Вместо этого мы хотим скомпилировать программу для четко определенной _целевой системы_.

## Установка Rust Nightly

У Rust есть 3 канала выпуска раст: _stable_, _beta_, и _nightly_. [The Rust Book] хорошо объясняет разницу между этими ветками, поэтому уделите пару минут [этому материалу][release-channels]. Для создания ОС нам нужны некоторые экспериментальные возможности, которые доступны только в nightly-режиме, поэтому нам необходимо установить nightly версию компилятора Rust.

[release-channels]: https://doc.rust-lang.ru/book/appendix-07-nightly-rust.html

Для этого я настоятельно рекомендую [rustup]. Он позволяет устанавливать nightly, beta, и stable компиляторы рядом друг с другом и легко обновлять. Чтобы использовать nightly компилятор для текущего проекта, выполните команду  `rustup override set nightly`. Как вариант, вы можете добавить файл `rust-toolchain` с содержимым `nightly` в корневую директорию проекта. Проверить, что nightly компилятор успешно установился, можно командой `rustc --version`: версия должна оканчиваться на `-nightly`.

[Rustup][rustup] делает лёгким изменение между различными каналами Rust, на глобальном или локальном для проекта уровне. По умолчанию устанавливается стабильная версия Rust. Для установки ночной версии выполните команду:

```console
$ rustup toolchain install nightly
```

Чтобы использовать nightly-компилятор в текущем проекте, необходимо выполнить команду:

```console
$ rustup override set nightly
```

[rustup]: https://www.rustup.rs/

Nightly-компилятор позволяет нам использовать некоторые экспериментальные возможности с помощью _feature флагов_ в начале файла. Например, нам нужно включить экспериментальный [`asm!` макрос]. Просто добавим `#![feature(asm)]` в начало `main.rs` и все заработает. Обратите внимание, что экспериментальные возможности полностью нестабильны, а значит в будущих версиях Rust они могут быть изменены или удалены без предварительного предупреждения. По этой причине мы будем использовать их только в случае крайней необходимости.

[`asm!` macro]: https://doc.rust-lang.org/unstable-book/library-features/asm.html

#### Собираем все вместе

Наш конфигурационный файл для цели выглядит так:

```json
//
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
    "linker-flavor": "ld.lld",
    "linker": "rust-lld",
    "panic-strategy": "abort",
    "disable-redzone": true,
    "features": "-mmx,-sse,+soft-float"
}
```

### Компиляция нашего ядра

Compiling for our new target will use Linux conventions (I'm not quite sure why, I assume that it's just LLVM's default). This means that we need an entry point named `_start` as described in the [previous post]:

При компиляции для нашей новой цели будут использованы соглашения, использующиеся в Linux. Это означает, что нам нужна точка входа с именем `_start`, как написано в [предыдущей главе].

[previous post]: @/edition-2/posts/01-freestanding-rust-binary/index.md

TODO перевести код

```rust
// src/main.rs

#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}
```

Обратите внимание, что точка входа должна называться `_start` независимо от вашей текущей ОС.

Теперь мы можем скомпилировать ядро для нашей новой цели, передав флаг `--target` с именем JSON-файла:

```
> cargo build --target x86_64-blog_os.json

error[E0463]: can't find crate for `core`
```

Мы получаем ошибку! Она сообщает нам, что компилятор Rust не может найти [библиотеку `core`]. Эта библиотека содержит базовые типы, такие как `Result`, `Option` и итераторы. Она неявно подключается во все `no_std` крейты.

[`core` library]: https://doc.rust-lang.org/nightly/core/index.html

Проблема в том, что библиотека `core` распространяется вместе с компилятором Rust как _перекомпилированная_ библиотека. Таким образом, она работает только с поддерживаемыми триплетами (например `x86_64-unknown-linux-gnu`), но не с пользовательскими. Чтобы исправить ошибку, нам нужно скомпилировать `core` библиотеку для нашей цели.

#### Опция `build-std`

That's where the [`build-std` feature] of cargo comes in. It allows to recompile `core` and other standard library crates on demand, instead of using the precompiled versions shipped with the Rust installation. This feature is very new and still not finished, so it is marked as "unstable" and only available on [nightly Rust compilers].

[`build-std` feature]: https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#build-std
[nightly Rust compilers]: #installing-rust-nightly

To use the feature, we need to create a [cargo configuration] file at `.cargo/config.toml` with the following content:

```toml
# in .cargo/config.toml

[unstable]
build-std = ["core", "compiler_builtins"]
```

This tells cargo that it should recompile the `core` and `compiler_builtins` libraries. The latter is required because it is a dependency of `core`. In order to recompile these libraries, cargo needs access to the rust source code, which we can install with `rustup component add rust-src`.

<div class="note">

**Note:** The `unstable.build-std` configuration key requires at least the Rust nightly from 2020-07-15.

</div>

After setting the `unstable.build-std` configuration key and installing the `rust-src` component, we can rerun the our build command:

```
> cargo build --target x86_64-blog_os.json
   Compiling core v0.0.0 (/…/rust/src/libcore)
   Compiling rustc-std-workspace-core v1.99.0 (/…/rust/src/tools/rustc-std-workspace-core)
   Compiling compiler_builtins v0.1.32
   Compiling blog_os v0.1.0 (/…/blog_os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.29 secs
```

We see that `cargo build` now recompiles the `core`, `rustc-std-workspace-core` (a dependency of `compiler_builtins`), and `compiler_builtins` libraries for our custom target.

#### Memory-Related Intrinsics Внутренние функции для работы с памятью

Компилятор Rust предполагает, что определенный набор функций доступен для всех систем. Большинство этих функций есть в только что скомпилированном крейте `compiler_builtins`. Однако некоторые функции, связанные с памятью, отсутствуют по умолчанию, потому что обычно они предоставляются стандартной библиотекой Си, которая установлена в системе. Например, такие функции, как `memset`, который устанавливает значение для всех байтов в блоке памяти заданное значение, `memcpy`, которая копирует один блок памяти в другой, `memcmp`, который сравнивает два блока памяти. Хотя нам и не требуются какие-либо из этих функций прямо сейчас для компиляции нашего ядра, они потребуются, как только мы добавить еще немного кода (например, при копирование структур).

Поскольку мы не можем использовать стандартную библиотеку Си, нам нужен альтернативный способ предоставить эти функции компилятору. Одним из возможных способов является собственная реализация функций, таких как `memset` и т. д., с использованием атрибута `#[no_mangle]` (чтобы избежать искажения имен компилятором). Однако это опасно, поскольку малейшая ошибка в реализации этих функций может привести к неопределенному поведению. Например, вы можете получить бесконечную рекурсию при реализации `memcpy` с использованием цикла `for`, потому что цикл `for` неявно вызывает метод [`IntoIterator::into_iter`] трейта, который может снова вызвать `memcpy` снова. Поэтому рекомендуется переиспользовать уже существующие, хорошо протестированные реализации.

[`IntoIterator::into_iter`]: https://doc.rust-lang.org/stable/core/iter/trait.IntoIterator.html#tymethod.into_iter

К счастью, крейт `compiler_builtins` уже содержит реализацию всех необходимых функций. Они просто отключены по умолчанию, чтобы не конфликтовать с функциями из стандартной библиотеки Си. Мы можем включить их, установив в настройках cargo флаг [`build-std-features`] в значение `["compiler-builtins-mem"]`. Как и флаг `build-std`, этот флаг может быть передан в командной строке как флаг `-Z` или настроен в таблице `unstable` в файле `.cargo/config.toml`. Поскольку мы хотим, чтобы этот флаг всегда использовался при сборке, мы настроим конфигурационный файл:

[`build-std-features`]: https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#build-std-features

```toml
# in .cargo/config.toml

[unstable]
build-std-features = ["compiler-builtins-mem"]
build-std = ["core", "compiler_builtins"]
```

(Support for the `compiler-builtins-mem` feature was only [added very recently](https://github.com/rust-lang/rust/pull/77284), so you need at least Rust nightly `2020-09-30` for it.)

Behind the scenes, this flag enables the [`mem` feature] of the `compiler_builtins` crate. The effect of this is that the `#[no_mangle]` attribute is applied to the [`memcpy` etc. implementations] of the crate, which makes them available to the linker.

[`mem` feature]: https://github.com/rust-lang/compiler-builtins/blob/eff506cd49b637f1ab5931625a33cef7e91fbbf6/Cargo.toml#L54-L55
[`memcpy` etc. implementations]: https://github.com/rust-lang/compiler-builtins/blob/eff506cd49b637f1ab5931625a33cef7e91fbbf6/src/mem.rs#L12-L69

With this change, our kernel has valid implementations for all compiler-required functions, so it will continue to compile even if our code gets more complex.
Теперь наше ядро содержит реализацию всех функций, необходимых компилятору в будущем.

## Установка цели по умолчанию

Каждый раз вручную вызывать компилятор с флагом `--target` слишком не удобно, поэтому мы попросим об этом cargo. Просто добавим новый параметр в наш [конфигурационный файл][cargo configuration] `.cargo/config.toml`:

[cargo configuration]: https://doc.rust-lang.org/cargo/reference/config.html

```toml
# в .cargo/config.toml

[build]
target = "x86_64-blog_os.json"
```

This tells `cargo` to use our `x86_64-blog_os.json` target when no explicit `--target` argument is passed. This means that we can now build our kernel with a simple `cargo build`. For more information on cargo configuration options, check out the [official documentation][cargo configuration].

Теперь cargo неявно будет передавать `--target x86_64-blog_os.json` компилятору, поэтому, чтобы собрать наше ядро, достаточно вызвать `cargo build`. Подробнее о `.cargo/config.toml` можно узнать в [официально документации][cargo configuration].

Теперь мы можем собрать наше ядро для системы без базовой ОС. Однако наша функция `_start`, которая будет вызвана загрузчиком, по-прежнему пуста. Пришло время вывести что-нибудь на экран.

## Вывод текста на экран

Самый простой способ вывести текст на экран — воспользоваться [текстовым VGA буфером][VGA text buffer]. Это специальная область памяти, которая содержит те данные, которые отображаются на экране. Обычно буфер состоит из 25 строк, каждая из которых содержит по 80 символьных ячеек. Каждая символьная ячейка отображает ASCII-символ некоторыми цветами переднего плана и фона. Вывод на экран выглядит так:

[VGA text buffer]: https://en.wikipedia.org/wiki/VGA-compatible_text_mode

![вывод на экран обычных ASCII-символов](https://upload.wikimedia.org/wikipedia/commons/f/f8/Codepage-437.png)

Мы обсудим точное строение VGA буфера в следующей главе, где мы напишем для него небольшой драйвер. Для вывода «Hello World!» нам просто нужно знать, что буфер расположен по адресу `0xb8000` и что каждая символьная ячейка состоит из байта ASCII и байта цвета.

Реализация выглядит так:

```rust
static HELLO: &[u8] = b"Hello World!";

#[no_mangle]
pub extern "C" fn _start() -> ! {
    let vga_buffer = 0xb8000 as *mut u8;
    
    for (i, &byte) in HELLO.iter().enumerate() {
        unsafe {
            *vga_buffer.offset(i as isize * 2) = byte;
            *vga_buffer.offset(i as isize * 2 + 1) = 0xb;
        }
    }

    loop {}
}
```

Сначала мы преобразуем число `0xb8000` в [сырой указатель][raw pointer]. Затем мы [перебираем][iterate] байты [статической][static] [байтовой строки][byte string] `HELLO`. Мы используем метод [`enumerate`] у итератора, чтобы получать в переменную `i` номер текущего шага (0, 1, ..., n). В теле цикла мы используем метод [`offset`], чтобы записать байт ASCII и байтом цвета (`0xb` это светло-голубой цвет).

[iterate]: https://doc.rust-lang.org/stable/book/ch13-02-iterators.html
[static]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#the-static-lifetime
[`enumerate`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.enumerate
[byte string]: https://doc.rust-lang.org/reference/tokens.html#byte-string-literals
[raw pointer]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer
[`offset`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.offset

Обратите внимание, что операции с памятью находятся в блоке [`unsafe`]. Дело в том, что компилятор Rust не может доказать, что созданный сырой указатель действителен. Он может указывать куда угодно, а операции над такими указателями могут приводить к повреждению данных в памяти. Используя блок `unsafe`, мы говорим компилятору, что абсолютно уверены в правильности этих операций. Заметим, что `unsafe` не отключает проверки безопасности Rust. Он позволяет вам делать только [пять дополнительных вещей][five additional things].

[`unsafe`]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html
[five additional things]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#unsafe-superpowers

Очень важно, что это **исключительный случай, который не соответствует идиоматическому Rust**. Использовать `unsafe` стоит только в случаях, когда вы уверены, что нет другого способа сделать желаемое. Кроме того, вы должны понимать, что делаете все правильно. Не забывайте про тестирование, все таки каждому человеку свойственно ошибаться.

Обратите внимание, что операции с памятью находятся в блоке [`unsafe`]. Дело в том, что компилятор Rust не может доказать, что созданный сырой указатель действителен. Он может указывать куда угодно, а операции над такими указателями могут приводить к повреждению данных в памяти. Используя блок `unsafe`, мы говорим компилятору, что абсолютно уверены в правильности этих операций. Заметим, что `unsafe` не отключает проверки безопасности Rust. Он позволяет вам делать только [пять дополнительных вещей][five additional things].

So we want to minimize the use of `unsafe` as much as possible. Rust gives us the ability to do this by creating safe abstractions. For example, we could create a VGA buffer type that encapsulates all unsafety and ensures that it is _impossible_ to do anything wrong from the outside. This way, we would only need minimal amounts of `unsafe` and can be sure that we don't violate [memory safety]. We will create such a safe VGA buffer abstraction in the next post.

Все же мы хотим минимизировать использование `unsafe` блоков настолько, насколько это возможно. Rust предлагает делать это с помощью создания безопасных абстракций. Например, мы можем создать свой тип буфера VGA, который инкапсулировал всю небезопасную логику и гарантировал, что _невозможно_ сделать что-то неправильно извне. Таким образом, нам понадобиться минимальное количество `unsafe` и мы можем быть уверены, что не нарушаем [безопасность памяти][memory safety]. В следующей главе мы создадим такую абстракцию буфера VGA.

[memory safety]: https://en.wikipedia.org/wiki/Memory_safety
