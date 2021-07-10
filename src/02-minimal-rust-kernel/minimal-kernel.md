## Небольшое ядро
Теперь, когда мы примерно знаем, как загружается компьютер, пришло время создать наше собственное небольшое ядро. Наша цель - создать образ диска, который выводит "Hello World!" на экран при загрузке. Для этого мы воспользуемся нашим [автономным бинарным файлом] из предыдущей главы.

Как вы помните, мы создали автономный бинарный файл с помощью `cargo`, но в зависимости от ОС нам потребовались разные имена точек входа и разные флаги компилятора. Дело в том, что по умолчанию `cargo` создает исполняемый файл для _хост-системы_, т. е. для системы, в которой вы сейчас работаете. Это не то, что мы хотим, потому что ядро на начальных стадиях загрузки компьютера, где еще нет никакой ОС. Вместо этого мы хотим скомпилировать программу для четко определенной _целевой системы_.

### Установка Rust Nightly
У Rust есть 3 release-ветки: _stable_, _beta_, и _nightly_. [The Rust Book] хорошо объясняет разницу между этими ветками, поэтому уделите пару минут [этому материалу](https://doc.rust-lang.org/book/appendix-07-nightly-rust.html#choo-choo-release-channels-and-riding-the-trains). Для создания ОС нам нужны некоторые экспериментальные возможности, которые доступны только в nightly-режиме, поэтому нам необходимо установить nightly версию компилятора Rust.

Для этого я настоятельно рекомендую [rustup]. Он позволяет устанавливать nightly, beta, и stable компиляторы рядом друг с другом и легко обновлять. Чтобы использовать nightly компилятор для текущего проекта, выполните команду  `rustup override set nightly`. Как вариант, вы можете добавить файл `rust-toolchain` с содержимым `nightly` в корневую директорию проекта. Проверить, что nightly компилятор успешно установился, можно командой `rustc --version`: версия должна оканчиваться на `-nightly`.

[rustup]: https://www.rustup.rs/

Nightly-компилятор позволяет нам использовать некоторые экспериментальные возможности с помощью _feature флагов_ в начале файла. Например, нам нужно включить экспериментальный [`asm!` макрос]. Просто добавим `#![feature(asm)]` в начало `main.rs` и все заработает. Обратите внимание, что экспериментальные возможности полностью нестабильны, а значит в будущих версиях Rust они могут быть изменены или удалены без предварительного предупреждения. По этой причине мы будем использовать их только в случае крайней необходимости.

[`asm!` macro]: https://doc.rust-lang.org/unstable-book/library-features/asm.html

### Настройка цели
Cargo поддерживает различные целевые системы. Для выбора таковой необходимо использовать параметр командной строки `--target`. Цель также называют _[target triple]_. Она
содержит в своем названии архитектуру процессора, поставщика, операционную систему и [ABI]. Например, цель с названием `x86_64-unknown-linux-gnu` описывает систему с `x86_64` CPU без четкого поставщика, с операционной системой Linux и GNU ABI. Rust поддерживает [большое количество разнообразных целей][platform-support], в том числе `arm-linux-androideabi` для Android, `wasm32-unknown-unknown` для [WebAssembly](https://www.hellorust.com/setup/wasm-target/).

[target triple]: https://clang.llvm.org/docs/CrossCompilation.html#target-triple
[ABI]: https://stackoverflow.com/a/2456882
[platform-support]: https://forge.rust-lang.org/release/platform-support.html
[custom-targets]: https://doc.rust-lang.org/nightly/rustc/targets/custom.html

Однако для нашей целевой системы нам требуются некоторые специальные параметры конфигурации (например, отсутствие базовой ОС), поэтому ни одна из [существующих целей][platform-support] нам не подходит. К счастью, Rust позволяет нам определить свою [собственную цель][custom-targets] с помощью JSON-файла. Например, JSON-файл для цели `x86_64-unknown-linux-gnu` выглядит так:

```json
{
    "llvm-target": "x86_64-unknown-linux-gnu",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "linux",
    "executables": true,
    "linker-flavor": "gcc",
    "pre-link-args": ["-m64"],
    "morestack": false
}
```

Большинство полей требуются LLVM для генерации кода для этой платформы. Например, поле [`data-layout`] определяет размер различных целочисленных типов, типов с плавающей запятой и указателей. Также есть поля, которые Rust использует для условной компиляции, например, поле `target-pointer-width`. Третий тип полей определяет, как должен быть скомпилирован крейт. Например, поле `pre-link-args` определяет аргументы, которые будут переданы [компоновщику][linker].

[`data-layout`]: https://llvm.org/docs/LangRef.html#data-layout
[linker]: https://en.wikipedia.org/wiki/Linker_(computing)

Мы будем компилировать наше ядро для `x86_64`, поэтому наше описание цели будет похоже на указанное выше. Начнем с создания файла `x86_64-blog_os.json` (можно выбрать любое имя) со следующим содержимым:

```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true
}
```

Обратите внимание, что мы изменили ОС в поле `llvm-target` и там, где раньше был `linux`, теперь `none`. 

Добавим еще два поля:


```json
"linker-flavor": "ld.lld",
"linker": "rust-lld",
```

Вместо использования компоновщика по умолчанию (который может не поддерживать цели Linux), мы используем кроссплатформенный компоновщик [LLD], поставляемый с Rust, для компоновки нашего ядра.

[LLD]: https://lld.llvm.org/

```json
"panic-strategy": "abort",
```

This setting specifies that the target doesn't support [stack unwinding] on panic, so instead the program should abort directly. This has the same effect as the `panic = "abort"` option in our Cargo.toml, so we can remove it from there. (Note that in contrast to the Cargo.toml option, this target option also applies when we recompile the `core` library later in this post. So be sure to add this option, even if you prefer to keep the Cargo.toml option.)


TODO перечитать
Параметр `panic-strategy` указывает, что цель не поддерживает [раскрутку стека] при панике, поэтому, вместо раскрутки, программа должна сразу завершать свою работу. Такой же эффект мы уже получали с помощью `panic = "abort"` в нашем `Cargo.toml`. Значит, что мы можем удалить `panic = "abort"` из конфигурационного файла. (Обратите внимание, что в отличие от параметра в `Cargo.toml`, `panic-strategy` применится и при компиляции `core` в этой главе.)

[stack unwinding]: https://www.bogotobogo.com/cplusplus/stackunwinding.php

```json
"disable-redzone": true,
```

Мы пишем ядро, поэтому в какой-то момент придется обрабатывать прерывания. Чтобы делать это безопасно, мы должны отключить определенную оптимизацию указателя стека под названием _"красная зона"_, потому что в противном случае это может привести к повреждению стека. О том, как отключить красную зону, можно узнать [здесь].

[disabling the red zone]: @/edition-2/posts/02-minimal-rust-kernel/disable-red-zone/index.md

```json
"features": "-mmx,-sse,+soft-float",
```

The `features` field enables/disables target features. We disable the `mmx` and `sse` features by prefixing them with a minus and enable the `soft-float` feature by prefixing it with a plus. Note that there must be no spaces between different flags, otherwise LLVM fails to interpret the features string.

Поле `features` включает и отключает функции, указываемые для цели. Мы отключаем `mmx` и `sse`, добавляя к ним знак минуса и включаем `soft-float` с помощью знака плюса. Обратите внимание, что между флагами не должно быть пробелов.

`mmx` и `sse` определяют поддержку инструкций [Single Instruction Multiple Data (SIMD)], которые часто могут значительно ускорить работу программы. Однако использование больших SIMD-регистров в ядре ОС приводит к проблемам с производительностью. Причина в том, что ядру необходимо приводить все регистры в исходное состояние перед продолжением прерванной программы. Это означает, что ядро должно сохранить полное SIMD-состояние в основной памяти при каждом вызове или аппаратном прерывании. Поскольку SIMD-состояние очень велико (512-1600 байт), а прерывания могут происходить очень часто, эти доп. операции сохранения и восстановления значительно снижают производительность. Чтобы этого избежать, мы отключаем SIMD для нашего ядра (но не для приложений, работающих поверх ядра!).

[Single Instruction Multiple Data (SIMD)]: https://en.wikipedia.org/wiki/SIMD

Отключая SIMD, возникает другая проблема. По умолчанию для операций с плавающей запятой на `x86_64` требуются SIMD-регистры. Поэтому мы добавляем `soft-float` функцию, которая эмулирует все операции с плавающей запятой с помощью программных функций, основанных на обычных целых числах.

For more information, see our post on [disabling SIMD](@/edition-2/posts/02-minimal-rust-kernel/disable-simd/index.md).

Подробнее об отключении SIMD можно узнать [здесь].

#### Собираем все вместе
Наш конфигурационный файл для цели выглядит так:

```json
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

The Rust compiler assumes that a certain set of built-in functions is available for all systems. Most of these functions are provided by the `compiler_builtins` crate that we just recompiled. However, there are some memory-related functions in that crate that are not enabled by default because they are normally provided by the C library on the system. These functions include `memset`, which sets all bytes in a memory block to a given value, `memcpy`, which copies one memory block to another, and `memcmp`, which compares two memory blocks. While we didn't need any of these functions to compile our kernel right now, they will be required as soon as we add some more code to it (e.g. when copying structs around).

Since we can't link to the C library of the operating system, we need an alternative way to provide these functions to the compiler. One possible approach for this could be to implement our own `memset` etc. functions and apply the `#[no_mangle]` attribute to them (to avoid the automatic renaming during compilation). However, this is dangerous since the slightest mistake in the implementation of these functions could lead to undefined behavior. For example, you might get an endless recursion when implementing `memcpy` using a `for` loop because `for` loops implicitly call the [`IntoIterator::into_iter`] trait method, which might call `memcpy` again. So it's a good idea to reuse existing well-tested implementations instead.

[`IntoIterator::into_iter`]: https://doc.rust-lang.org/stable/core/iter/trait.IntoIterator.html#tymethod.into_iter

Fortunately, the `compiler_builtins` crate already contains implementations for all the needed functions, they are just disabled by default to not collide with the implementations from the C library. We can enable them by setting cargo's [`build-std-features`] flag to `["compiler-builtins-mem"]`. Like the `build-std` flag, this flag can be either passed on the command line as `-Z` flag or configured in the `unstable` table in the `.cargo/config.toml` file. Since we always want to build with this flag, the config file option makes more sense for us:

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

#### Установка цели по умолчанию

To avoid passing the `--target` parameter on every invocation of `cargo build`, we can override the default target. To do this, we add the following to our [cargo configuration] file at `.cargo/config.toml`:

[cargo configuration]: https://doc.rust-lang.org/cargo/reference/config.html

```toml
# in .cargo/config.toml

[build]
target = "x86_64-blog_os.json"
```

This tells `cargo` to use our `x86_64-blog_os.json` target when no explicit `--target` argument is passed. This means that we can now build our kernel with a simple `cargo build`. For more information on cargo configuration options, check out the [official documentation][cargo configuration].

We are now able to build our kernel for a bare metal target with a simple `cargo build`. However, our `_start` entry point, which will be called by the boot loader, is still empty. It's time that we output something to screen from it.

### Вывод текста на экран
The easiest way to print text to the screen at this stage is the [VGA text buffer]. It is a special memory area mapped to the VGA hardware that contains the contents displayed on screen. It normally consists of 25 lines that each contain 80 character cells. Each character cell displays an ASCII character with some foreground and background colors. The screen output looks like this:

[VGA text buffer]: https://en.wikipedia.org/wiki/VGA-compatible_text_mode

![screen output for common ASCII characters](https://upload.wikimedia.org/wikipedia/commons/f/f8/Codepage-437.png)

We will discuss the exact layout of the VGA buffer in the next post, where we write a first small driver for it. For printing “Hello World!”, we just need to know that the buffer is located at address `0xb8000` and that each character cell consists of an ASCII byte and a color byte.

The implementation looks like this:

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

First, we cast the integer `0xb8000` into a [raw pointer]. Then we [iterate] over the bytes of the [static] `HELLO` [byte string]. We use the [`enumerate`] method to additionally get a running variable `i`. In the body of the for loop, we use the [`offset`] method to write the string byte and the corresponding color byte (`0xb` is a light cyan).

[iterate]: https://doc.rust-lang.org/stable/book/ch13-02-iterators.html
[static]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#the-static-lifetime
[`enumerate`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.enumerate
[byte string]: https://doc.rust-lang.org/reference/tokens.html#byte-string-literals
[raw pointer]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer
[`offset`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.offset

Note that there's an [`unsafe`] block around all memory writes. The reason is that the Rust compiler can't prove that the raw pointers we create are valid. They could point anywhere and lead to data corruption. By putting them into an `unsafe` block we're basically telling the compiler that we are absolutely sure that the operations are valid. Note that an `unsafe` block does not turn off Rust's safety checks. It only allows you to do [five additional things].

[`unsafe`]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html
[five additional things]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#unsafe-superpowers

I want to emphasize that **this is not the way we want to do things in Rust!** It's very easy to mess up when working with raw pointers inside unsafe blocks, for example, we could easily write beyond the buffer's end if we're not careful.

So we want to minimize the use of `unsafe` as much as possible. Rust gives us the ability to do this by creating safe abstractions. For example, we could create a VGA buffer type that encapsulates all unsafety and ensures that it is _impossible_ to do anything wrong from the outside. This way, we would only need minimal amounts of `unsafe` and can be sure that we don't violate [memory safety]. We will create such a safe VGA buffer abstraction in the next post.

[memory safety]: https://en.wikipedia.org/wiki/Memory_safety