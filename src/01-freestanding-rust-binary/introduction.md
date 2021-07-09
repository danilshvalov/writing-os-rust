+++
title = "A Freestanding Rust Binary"
weight = 1
path = "freestanding-rust-binary"
date = 2018-02-10

[extra]
chapter = "Bare Bones"
+++

Первый шаг в разработке нашего собственного ядра операционной системы - создание исполняемого файла Rust, который не использует стандартную библиотеку. Это позволяет запускать код Rust на [голом железе] (bare metal) без базовой операционной системы.

[голом железе]: https://en.wikipedia.org/wiki/Bare_machine

<!-- more -->

Этот блог открыто разрабатывается на [Github]. Если у вас есть какие-то проблемы или вопросы, пожалуйста, откройте issue здесь. Вы также можете оставлять комментарии [внизу]. Весь исходный код с этой статьи можно найти в [`post-01`].

[GitHub]: https://github.com/phil-opp/blog_os
[внизу]: #comments
[post branch]: https://github.com/phil-opp/blog_os/tree/post-01

<!-- toc -->

## Введение

Чтобы написать ядро операционной системы, нам нужен код, который не зависит от каких-либо функций операционной системы. Это означает, что мы не можем использовать потоки (threads), файлы (files), память в куче (heap memory), сеть (the network), случайные числа (random numbers), стандартный вывод (standard output) или любые другие функции, требующие абстракции со стороны операционной системы (OS abstractions) или определенного оборудования. В этом есть смысл, поскольку мы пытаемся написать собственную операционную систему и собственные драйверы.

Это означает, что мы не можем использовать большую часть [стандартной библиотеки Rust], но есть много возможностей (features) из Rust, которые мы _можем_ использовать. Например, мы можем использовать [итераторы], [замыкания], [pattern matching], [option] и [result], [форматирование строк] и, конечно же, [систему владения]. Эти возможности позволяют написать ядро очень выразительным и высокоуровневым способом, не беспокоясь о [неопределенном поведении] (undefined behavior) или о [безопасности памяти] (memory safety).

[option]: https://doc.rust-lang.org/core/option/
[result]:https://doc.rust-lang.org/core/result/
[стандартной библиотеки Rust]: https://doc.rust-lang.org/std/
[итераторы]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[замыкания]: https://doc.rust-lang.org/book/ch13-01-closures.html
[pattern matching]: https://doc.rust-lang.org/book/ch06-00-enums.html
[форматирование строк]: https://doc.rust-lang.org/core/macro.write.html
[систему владения]: https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html
[неопределенном поведении]: https://www.nayuki.io/page/undefined-behavior-in-c-and-cplusplus-programs
[безопасности памяти]: https://tonyarcieri.com/it-s-time-for-a-memory-safety-intervention

Для разработки ядра ОС на Rust нам необходимо создать исполняемый файл, который можно запускать без операционной системы. Такой файл обычно называют “автономный” (freestanding) или “голый” (bare-metal).

В этой статье описываются шаги, необходимые для создания автономного бинарника на Rust, и объясняется, почему эти шаги необходимы. Если вас интересует краткая сводка, вы можете **[перейти к резюме](#summary)**



## Реализация Panic

Атрибут `panic_handler` определяет функцию, которую компилятор должен вызвать при возникновении [паники] (panic) входе работы программы. Стандартная библиотека предоставляет собственную функцию обработки паники, но так как мы используем `no_std`, нам нужно определить ее самостоятельно:

[паники]: https://doc.rust-lang.org/stable/book/ch09-01-unrecoverable-errors-with-panic.html

```rust
// in main.rs

use core::panic::PanicInfo;

/// Эта функция вызывается при панике
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

[Параметр `PanicInfo`][PanicInfo] содержит в себе информацию о файле и строку, где произошла паника, а также может содержать сообщение с ошибкой. Функция должна никогда не возвращать результат (never return), поэтому она помечена как [расходящаяся функция] (diverging function) с помощью возврата [“пустого” типа] `!`. Сейчас мы мало что можем сделать с этой функцией, поэтому просто создадим бесконечный цикл.

[PanicInfo]: https://doc.rust-lang.org/nightly/core/panic/struct.PanicInfo.html
[расходящаяся функция]: https://doc.rust-lang.org/1.30.0/book/first-edition/functions.html#diverging-functions
[“пустого” типа]: https://doc.rust-lang.org/nightly/std/primitive.never.html

## Элемент языка `eh_personality`

Элементы языка - это специальные функции и типы, которые требует компилятор. Например, трейт [`Copy`] - это элемент языка, который сообщает компилятору какие типы имею [_семантику копирования_][`Copy`]. Если посмотреть на [реализацию][copy code], мы увидим, что трейт помечен специальным атрибутом `#[lang = "copy"]`, который определяет его как языковой элемент.

[`Copy`]: https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html
[copy code]: https://github.com/rust-lang/rust/blob/485397e49a02a3b7ff77c17e4a3f16c653925cb3/src/libcore/marker.rs#L296-L299

Хотя создание пользовательских реализаций языковых элементов возможно, делать это следует только в крайнем случае. Причина в том, что языковые элементы представляют собой очень нестабильные детали реализации, типы которых не проверяются компилятором статически. К счастью, есть более безопасный способ исправить указанную выше ошибку.

[Элемент языка `eh_personality`] помечает функцию, которая используется для реализации [раскрутки стека]. По умолчанию Rust использует раскрутку стека для запуска деструкторов всех переменных, живущих на стеке, в случае паники. Это гарантирует, что вся используемая память будет освобождена, и позволяет родительскому потоку перехватить панику и продолжить выполнение. Однако раскрутка стека - это сложный процесс, требующий наличие библиотек, специфичных для ОС (например [libunwind] на Linux или [structured exception handling] на Windows), так что мы не будем использовать раскрутку в нашей операционной системе.

[Элемент языка `eh_personality`]: https://github.com/rust-lang/rust/blob/edb368491551a77d77a48446d4ee88b35490c565/src/libpanic_unwind/gcc.rs#L11-L45
[раскрутки стека]: https://www.bogotobogo.com/cplusplus/stackunwinding.php
[libunwind]: https://www.nongnu.org/libunwind/
[structured exception handling]: https://docs.microsoft.com/de-de/windows/win32/debug/structured-exception-handling

### Отключение раскрутки стека

Бывают случаи, когда раскрутка нежелательна, поэтому Rust предоставляет возможность [прекратить работу исполняемого файла] при панике. Это отключает генерацию символов, необходимых для раскрутки стека, и, таким образом, значительно уменьшает размер бинарного файла. Есть несколько способов отключения раскрутки. Самый простой - добавить несколько строк в наш `Cargo.toml`:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```
 
 Мы указываем прекратить работу исполняемого файла при панике для `dev` профиля (используется при `cargo build`) и для `release` профиля (используется при `cargo build --release`). Теперь языковой элемент `eh_personality` больше не требуется.

[прекратить работу исполняемого файла]: https://github.com/rust-lang/rust/pull/32900

Теперь мы исправили обе указанные выше ошибки. Однако, если мы попытаемся скомпилировать программу, возникнет другая ошибка:

```
> cargo build
error: requires `start` lang_item
```

Компилятор не может найти языковой элемент `start`, который определяет точку входа в программу.

## Аттрибут `start`

One might think that the `main` function is the first function called when you run a program. However, most languages have a [runtime system], which is responsible for things such as garbage collection (e.g. in Java) or software threads (e.g. goroutines in Go). This runtime needs to be called before `main`, since it needs to initialize itself.

Можно подумать, что функция `main` - это первая функция, которая будет вызвана при запуске вашей программы. Однако у большинства языков программирования есть [среда выполнения] (runtime system), которая отвечает за такие вещи, как сборка мусора (например в Java) или программные потоки (например goroutines в Go). Среду выполнения нужно вызвать перед `main`, поскольку она должна инициализировать себя.

[среда выполнения]: https://en.wikipedia.org/wiki/Runtime_system

В обычном бинарном файле Rust, который подключает стандартную библиотеку, выполнение начинается в runtime-библиотеке языка Си под названием `crt0` (“C runtime zero”), которая устанавливает среду для Си-приложения. Это включает в себя создание стека и размещение аргументов в правильных регистрах. Затем среда выполнения Си вызывает [точку входа среды выполнения Rust][rt::lang_start], которая помечена языковым элементом `start`. Rust имеет очень маленькую среду выполнения, которая заботится о некоторых мелочах, таких как настройка защиты от переполнения стека или печать backtrace при панике. Затем среда выполнения вызывает функцию `main`.

[rt::lang_start]: https://github.com/rust-lang/rust/blob/bb4d1491466d8239a7a5fd68bd605e3276e97afb/src/libstd/rt.rs#L32-L73

Наш автономный исполняемый файл не имеет доступа к среде выполнения Rust и к `crt0`, поэтому нам нужно определить нашу собственную точку входа. Реализация языкового элемента `start` не решит нашу проблему, так как для работы потребуется `crt0`. Вместо этого нам нужно напрямую перезаписать точку входа `crt0`.

### Перезаписывание точки входа
Чтобы сообщить компилятору Rust, что мы не хотим использовать обычную цепочку точек входа, мы должны использовать `#![no_main]` атрибут.

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

/// Эта функция вызывается при панике
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

Вы можете заметить, что мы удалили функцию `main`. Дело в том, что функция `main` не имеет смысла без среды выполнения, которая её вызывает. Вместо этого теперь мы перезаписываем точку входа в операционную систему с помощью функции `_start`:


```rust
#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

Используя атрибут `#[no_mangle]`, мы отключаем [искажение имен] (name mangling), чтобы гарантировать, что компилятор Rust создаст функцию с именем `_start`. Без этого атрибута компилятор сгенерирует какой-нибудь загадочный символ, наподобие `_ZN3blog_os4_start7hb173fedf945531caE`, чтобы дать каждой функции уникальное имя. Атрибут является обязательным, поскольку на следующем шаге нам нужно сообщить имя функции точки входа компоновщику.

Мы также должны пометить функцию как `extern "C"`, чтобы сообщить компилятору, что он должен использовать [соглашение о вызове языка Си] (C calling convention) для этой функции (вместо неопределенного соглашения о вызове языка Rust). Причина, по которой мы назвали точку входа именем `_start`, заключается в том, что это имя точки входа по умолчанию для большинства систем. 

[искажение имен]: https://en.wikipedia.org/wiki/Name_mangling
[Cсоглашение о вызове языка Си]: https://en.wikipedia.org/wiki/Calling_convention

Возвращаемый тип `!` говорит о том, что функция помечена как расходящаяся (diverging), то есть никогда не возвращает результат. Это необходимо, потому что точка входа не вызывается какой-либо функцией, но вызывается непосредственно операционной системой или загрузчиком (bootloader). Поэтому вместо возврата значений точка входа должна, например, вызывать [системную функцию `exit`]. В нашем случае выключение устройства может быть разумным действием, так как больше ничего не остается делать, если исполняемый файл завершает свою работу. На данный момент мы выполняем требование, создавая бесконечный цикл.

[системную функцию `exit`]: https://en.wikipedia.org/wiki/Exit_(system_call)

Если запустить `cargo build` теперь, то мы получим ужасную ошибку _компоновщика_.

## Ошибки компоновщика

Компоновщик - это программа, которая упаковывает сгенерированный код в исполняемый файл. Поскольку формат исполняемый файлов в Linux, Windows и macOS различный, в каждой ОС есть собственный компоновщик. Так как компоновщики разные, то и сообщения об ошибке будут отличаться. Но причина ошибок одинаковая: конфигурация компоновщика по умолчанию предполагает, что наша программа зависит от среды выполнения Си, хотя это не так.

Для устранения ошибки мы должны сообщить компоновщику, что он не должен подключать среду исполнения Си. Мы можем сделать это передав определенный набор аргументов компоновщику или скомпилировать наш файл для ”голого железа”.

### Компиляция для ”голого железа”

По умолчанию Rust пытается создать исполняемый файл, который можно запустить на текущей системе. Например, если вы используете Windows на архитектуре `x86_64`, то Rust пытается создать `.exe` файл, который использует `x86_64` инструкции. Текущую операционную систему также называют `host system`.

Для описания различных сред исполнения Rust использует строку называемую [_target triple_]. Вы можете увидеть `target triple` для вашей системы с помощью команды `rustc --version --verbose`:

[_target triple_]: https://clang.llvm.org/docs/CrossCompilation.html#target-triple

```
rustc 1.35.0-nightly (474e7a648 2019-04-07)
binary: rustc
commit-hash: 474e7a6486758ea6fc761893b1a49cd9076fb0ab
commit-date: 2019-04-07
host: x86_64-unknown-linux-gnu
release: 1.35.0-nightly
LLVM version: 8.0
```

Приведенный выше вывод получен из `x86_64` Linux системы. Мы видим, что `host` система - это `x86_64-unknown-linux-gnu`. Это означает, что система использует `x86_64` архитектуру, поставщиком ОС является  `unknown`, операционная система - `linux`, а [ABI] - (`gnu`).

[ABI]: https://en.wikipedia.org/wiki/Application_binary_interface

При компиляции для нашей основной среды, компилятор Rust и компоновщик предполагали, что существует базовая операционная система, такая как Linux или Windows, использующая среду выполнения Си по умолчанию, что привело к ошибкам компоновщика. Чтобы избежать ошибки компоновщика, мы скомпилируем исполняемый файл для другой среды, которая не требует базовой ОС.

Примером таковой является среда "голого железа", именуемая как `thumbv7em-none-eabihf`, которая описывает [встроенную] [ARM] систему. Детали нам не важны, самое главное только то, что target triple не имеет базовой ОС, на что указывает `none` в названии target triple. Чтобы иметь возможность компилировать для этого target, мы также должны добавить его в rustup:

[embedded]: https://en.wikipedia.org/wiki/Embedded_system
[ARM]: https://en.wikipedia.org/wiki/ARM_architecture

```
rustup target add thumbv7em-none-eabihf
```

Эта команда загрузит копию стандартной библиотеки и ядро языка для системы. Теперь мы можем создать наш автономный исполняемый файл для этого target'a;

```
cargo build --target thumbv7em-none-eabihf
```

Используя аргумент `--target` мы [кросс компилируем] наш исполняемый файл для bare metal target system. Поскольку target system не имеет операционной системы, компоновщик не пытается подключить среду выполнения Си и создание нашего исполняемого файла происходит успешно без ошибок компоновщика.

[кросс компилируем]: https://en.wikipedia.org/wiki/Cross_compiler

This is the approach that we will use for building our OS kernel. Instead of `thumbv7em-none-eabihf`, we will use a [custom target] that describes a `x86_64` bare metal environment. The details will be explained in the next post.

[custom target]: https://doc.rust-lang.org/rustc/targets/custom.html

### Linker Arguments

Instead of compiling for a bare metal system, it is also possible to resolve the linker errors by passing a certain set of arguments to the linker. This isn't the approach that we will use for our kernel, therefore this section is optional and only provided for completeness. Click on _"Linker Arguments"_ below to show the optional content.

<details>

<summary>Linker Arguments</summary>

In this section we discuss the linker errors that occur on Linux, Windows, and macOS, and explain how to solve them by passing additional arguments to the linker. Note that the executable format and the linker differ between operating systems, so that a different set of arguments is required for each system.

#### Linux

On Linux the following linker error occurs (shortened):

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x12): undefined reference to `__libc_csu_fini'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x19): undefined reference to `__libc_csu_init'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x25): undefined reference to `__libc_start_main'
          collect2: error: ld returned 1 exit status
```

The problem is that the linker includes the startup routine of the C runtime by default, which is also called `_start`. It requires some symbols of the C standard library `libc` that we don't include due to the `no_std` attribute, therefore the linker can't resolve these references. To solve this, we can tell the linker that it should not link the C startup routine by passing the `-nostartfiles` flag.

One way to pass linker attributes via cargo is the `cargo rustc` command. The command behaves exactly like `cargo build`, but allows to pass options to `rustc`, the underlying Rust compiler. `rustc` has the `-C link-arg` flag, which passes an argument to the linker. Combined, our new build command looks like this:

```
cargo rustc -- -C link-arg=-nostartfiles
```

Now our crate builds as a freestanding executable on Linux!

We didn't need to specify the name of our entry point function explicitly since the linker looks for a function with the name `_start` by default.

#### Windows

On Windows, a different linker error occurs (shortened):

```
error: linking with `link.exe` failed: exit code: 1561
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1561: entry point must be defined
```

The "entry point must be defined" error means that the linker can't find the entry point. On Windows, the default entry point name [depends on the used subsystem][windows-subsystems]. For the `CONSOLE` subsystem the linker looks for a function named `mainCRTStartup` and for the `WINDOWS` subsystem it looks for a function named `WinMainCRTStartup`. To override the default and tell the linker to look for our `_start` function instead, we can pass an `/ENTRY` argument to the linker:

[windows-subsystems]: https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol

```
cargo rustc -- -C link-arg=/ENTRY:_start
```

From the different argument format we clearly see that the Windows linker is a completely different program than the Linux linker.

Now a different linker error occurs:

```
error: linking with `link.exe` failed: exit code: 1221
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1221: a subsystem can't be inferred and must be
          defined
```

This error occurs because Windows executables can use different [subsystems][windows-subsystems]. For normal programs they are inferred depending on the entry point name: If the entry point is named `main`, the `CONSOLE` subsystem is used, and if the entry point is named `WinMain`, the `WINDOWS` subsystem is used. Since our `_start` function has a different name, we need to specify the subsystem explicitly:

```
cargo rustc -- -C link-args="/ENTRY:_start /SUBSYSTEM:console"
```

We use the `CONSOLE` subsystem here, but the `WINDOWS` subsystem would work too. Instead of passing `-C link-arg` multiple times, we use `-C link-args` which takes a space separated list of arguments.

With this command, our executable should build successfully on Windows.

#### macOS

On macOS, the following linker error occurs (shortened):

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: entry point (_main) undefined. for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

This error message tells us that the linker can't find an entry point function with the default name `main` (for some reason all functions are prefixed with a `_` on macOS). To set the entry point to our `_start` function, we pass the `-e` linker argument:

```
cargo rustc -- -C link-args="-e __start"
```

The `-e` flag specifies the name of the entry point function. Since all functions have an additional `_` prefix on macOS, we need to set the entry point to `__start` instead of `_start`.

Now the following linker error occurs:

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: dynamic main executables must link with libSystem.dylib
          for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

macOS [does not officially support statically linked binaries] and requires programs to link the `libSystem` library by default. To override this and link a static binary, we pass the `-static` flag to the linker:

[does not officially support statically linked binaries]: https://developer.apple.com/library/archive/qa/qa1118/_index.html

```
cargo rustc -- -C link-args="-e __start -static"
```

This still does not suffice, as a third linker error occurs:

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: library not found for -lcrt0.o
          clang: error: linker command failed with exit code 1 […]
```

This error occurs because programs on macOS link to `crt0` (“C runtime zero”) by default. This is similar to the error we had on Linux and can be also solved by adding the `-nostartfiles` linker argument:

```
cargo rustc -- -C link-args="-e __start -static -nostartfiles"
```

Now our program should build successfully on macOS.

#### Unifying the Build Commands

Right now we have different build commands depending on the host platform, which is not ideal. To avoid this, we can create a file named `.cargo/config.toml` that contains the platform specific arguments:

```toml
# in .cargo/config.toml

[target.'cfg(target_os = "linux")']
rustflags = ["-C", "link-arg=-nostartfiles"]

[target.'cfg(target_os = "windows")']
rustflags = ["-C", "link-args=/ENTRY:_start /SUBSYSTEM:console"]

[target.'cfg(target_os = "macos")']
rustflags = ["-C", "link-args=-e __start -static -nostartfiles"]
```

The `rustflags` key contains arguments that are automatically added to every invocation of `rustc`. For more information on the `.cargo/config.toml` file check out the [official documentation](https://doc.rust-lang.org/cargo/reference/config.html).

Now our program should be buildable on all three platforms with a simple `cargo build`.

#### Should You Do This?

While it's possible to build a freestanding executable for Linux, Windows, and macOS, it's probably not a good idea. The reason is that our executable still expects various things, for example that a stack is initialized when the `_start` function is called. Without the C runtime, some of these requirements might not be fulfilled, which might cause our program to fail, e.g. through a segmentation fault.

If you want to create a minimal binary that runs on top of an existing operating system, including `libc` and setting the `#[start]` attribute as described [here](https://doc.rust-lang.org/1.16.0/book/no-stdlib.html) is probably a better idea.

</details>

## Summary

A minimal freestanding Rust binary looks like this:

`src/main.rs`:

```rust
#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}

/// Эта функция вызывается при панике
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`Cargo.toml`:

```toml
[package]
name = "crate_name"
version = "0.1.0"
authors = ["Author Name <author@example.com>"]

# the profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# the profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic
```

To build this binary, we need to compile for a bare metal target such as `thumbv7em-none-eabihf`:

```
cargo build --target thumbv7em-none-eabihf
```

Alternatively, we can compile it for the host system by passing additional linker arguments:

```bash
# Linux
cargo rustc -- -C link-arg=-nostartfiles
# Windows
cargo rustc -- -C link-args="/ENTRY:_start /SUBSYSTEM:console"
# macOS
cargo rustc -- -C link-args="-e __start -static -nostartfiles"
```

Note that this is just a minimal example of a freestanding Rust binary. This binary expects various things, for example that a stack is initialized when the `_start` function is called. **So for any real use of such a binary, more steps are required**.

## Что дальше?

В [следующей статье] объясняются шаги, необходимые для превращения нашего автономного бинарного файла в небольшое ядро операционной системы. Это включает в себя создание настраиваемой цели (target), объединение нашего исполняемого файла с загрузчиком и обучение тому, как выводить что-нибудь на экран.

[next post]: @/edition-2/posts/02-minimal-rust-kernel/index.md