# Выход из QEMU

Прямо сейчас у нас есть бесконечный цикл в конце функции `_start`, поэтому после тестирования нам нужно вручную закрывать QEMU. Возможным и весьма неплохим решением будет реализация выключения нашей ОС после окончания тестов. К сожалению сделать это не так просто, потому что нам пришлось бы реализовать [APM] и [ACPI] стандарты управления питанием.

[APM]: https://wiki.osdev.org/APM
[ACPI]: https://wiki.osdev.org/ACPI

К счастью, есть другой выход: QEMU поддерживает специальное устройство выхода `isa-debug-exit`, которое обеспечивает простой способ завершения работы QEMU. Чтобы использовать его, нам нужно передать флаг `-device`. Давайте попросим cargo делать это за нас, просто добавив пару строк в  `Cargo.toml`:

```toml
# в Cargo.toml

[package.metadata.bootimage]
test-args = ["-device", "isa-debug-exit,iobase=0xf4,iosize=0x04"]
```

По умолчанию `bootimage runner` передает QEMU указанные в `test-args` аргументы при каждом запуске тестов. При обычном же запуске `cargo run` аргументы игнорируются.

Вместе с именем устройства `isa-debug-exit` мы передаем два параметра: `iobase` и `iosize`. Они устанавливают порт ввода/вывода, через который можно получить доступ к устройству из нашего ядра.

## Порты ввода/вывода

На архитектуре x86 существует два подхода для обмена данными между CPU и периферийным оборудованием: **ввода/вывод через память** и **ввода/вывод через порты**. Мы уже использовали ввод-вывод с использованием памяти для доступа к VGA-буферу по адресу `0xb8000`. Этот адрес не находится в ОЗУ, а в VGA-устройстве.

При вводе/выводе через порты для связи используется отдельная шина. Каждое подключенное периферийное устройство имеет свой набор портов. Для такого типа связи у процессора существуют специальные инструкции, — `in` и `out` — которые принимают номер порта и байт данных. Существуют также варианты, которые позволяют отправлять по 2 или по 4 байта за раз.

Устройство `isa-debug-exit` использует порты для ввода и вывода. Параметр `iobase` указывает адрес, на котором должно находится устройство (`0xf4` [обычно не используется][x86-io-ports]), а `iosize` устанавливает размер порта (`0x04` = 4 байта).

[x86-io-ports]: https://wiki.osdev.org/I/O_Ports#The_list

## Использование устройства выхода

Логика работы `isa-debug-exit` очень проста. Когда мы записываем `value` в порт, назначенный параметром `iobase`, QEMU завершает работу с [кодом возврата], который строится из выражения `(value << 1) | 1`. Записав значение `0`, QEMU завершит работу с кодом `(0 << 1) | 1 = 1`, запишем `1` — получим `(1 << 1) | 1 = 3`.

[exit-status]: https://ru.wikipedia.org/wiki/Код_возврата

Вместо того, чтобы вручную вызывать ассемблерные инструкции `in` и `out`, мы воспользуемся абстракцией, предоставляемой крейтом [`x86_64`]. Давайте добавим этот крейт в зависимости нашего проекта:

[`x86_64`]: https://docs.rs/x86_64/0.14.2/x86_64/

```toml
# в Cargo.toml

[dependencies]
x86_64 = "0.14.2"
```

Теперь мы можем использовать [`Port`] из крейта [`x86_64`], чтобы создать функцию `exit_qemu`:

[`Port`]: https://docs.rs/x86_64/0.14.2/x86_64/instructions/port/struct.Port.html

```rust
// в src/main.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u32)]
pub enum QemuExitCode {
    Success = 0x10,
    Failed = 0x11,
}

pub fn exit_qemu(exit_code: QemuExitCode) {
    use x86_64::instructions::port::Port;

    unsafe {
        let mut port = Port::new(0xf4);
        port.write(exit_code as u32);
    }
}
```

Функция создает новый [`Port`] по адресу `0xf4`, который мы указали вместе с `iobase`. Затем мы записываем переданный код возврата в порт. Так как мы указали `iosize` равный 4 байтов, нам нужно привести наш `QemuExitCode` к `u32`. Обе операции должны быть заключены в блок `unsafe`, поскольку запись в порт ввода/вывода может привести к произвольному поведению.

Для представления кода возврата мы создали перечисление `QemuExitCode`. Теперь, если все тесты прошли успешно, мы возвращаем `Success` код, если же хотя бы один упал — код возврата `Failed`. `QemuExitCode` помечено атрибутом `#[repr(u32)]`, чтобы варианты перечисления размещались в памяти также, как и `u32`. По сути, сами значения кодов возврата не играют никакой роли, если они не конфликтуют с таковыми в QEMU. Например, использование `0` — не очень удачная идея, поскольку в итоге ноль превращается в единицу: `(0 << 1) | 1 = 1`. По умолчанию `1` используется в QEMU для обозначения ошибки во время запуска, поэтому мы не сможем отличить ошибку QEMU от успешного прохождения всех тестов.

Давайте обновим наш `test_runner`, чтобы завершать работу QEMU после прохождения всех тестов:

```rust
// в src/main.rs

fn test_runner(tests: &[&dyn Fn()]) {
    println!("Running {} tests", tests.len());
    for test in tests {
        test();
    }
    /// новое
    exit_qemu(QemuExitCode::Success);
}
```

Запустите `cargo test` теперь и вы увидите. что QEMU сразу закрывается после выполнения всех тестов. Проблема в том, что `cargo test` интерпретирует тест как не удачный, даже если мы передадим код возврата `Success`:

```console
$ cargo test
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running target/x86_64-blog_os/debug/deps/blog_os-5804fc7d2dd4c9be
Building bootloader
   Compiling bootloader v0.5.3 (/home/philipp/Documents/bootloader)
    Finished release [optimized + debuginfo] target(s) in 1.07s
Running: `qemu-system-x86_64 -drive format=raw,file=/…/target/x86_64-blog_os/debug/
    deps/bootimage-blog_os-5804fc7d2dd4c9be.bin -device isa-debug-exit,iobase=0xf4,
    iosize=0x04`
error: test failed, to rerun pass '--bin blog_os'
```

Дело в том, что `cargo test` рассматривает все коды возврата, не совпадающие с `0`, как ошибочные.

## Настройка кодов возврата

Чтобы обойти это ограничение, `bootimage` предоставляет ключ `test-success-exit-code` на этот случай. Он подменяет выбранный код возврата на `0`, чтобы cargo переварил его:

```toml
# в Cargo.toml

[package.metadata.bootimage]
test-args = […]
test-success-exit-code = 33         # (0x10 << 1) | 1
```

После этого `cargo test` должен корректно распознавать пройденные и упавшие тесты.

Теперь наш test runner автоматически закрывает QEMU и корректно сообщает о результатах тестирования. Мы по-прежнему видим, что QEMU открывается на небольшой промежуток времени, но этого недостаточно для чтения результатов. Было бы неплохо выводить результаты в консоль, а не в QEMU.

## Выводим результаты в консоль

Чтобы выводить результаты тестирования в консоль, нам нужно каким-то образом отправлять данные из ядра в систему, из которой вы запускаете QEMU. Этого можно добиться разными способами, например, посылать данные по протоколу TCP. Однако настройка сетевого стека — занятие не из простых, поэтому мы выберем более просто решение.

### Последовательный порт

Самый простой способ отправить данные — использовать [последовательный порт][serial port]. Это старый стандарт интерфейса, который большое не встречается в современных компьютерах. Его легко запрограммировать, к тому же QEMU умеет перенаправлять байты из последовательного порта в стандартный вывод системы или в файл.

[serial port]: https://en.wikipedia.org/wiki/Serial_port

Микросхемы, реализующие последовательный порт, называются [UART]. На архитектуре x86 существует немало [моделей UART][UART-models], но, к счастью, различия между ними небольшие: в основном, это некоторые дополнительные функции, которые нас не интересуют. Все распространенные UART совместимы с [16550 UART], поэтому мы будем опираться на эту именно модель.

[UART]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter
[UART-models]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter#UART_models
[16550 UART]: https://en.wikipedia.org/wiki/16550_UART

Давайте используем крейт [`uart_16550`] для инициализации UART и отправки данных через последовательный порт. Для этого добавим его в зависимости нашего проекта:

[`uart_16550`]: https://docs.rs/uart_16550

```toml
# в Cargo.toml

[dependencies]
uart_16550 = "0.2.0"
```

Крейт `uart_16550` предоставляет `SerialPort` для абстракции над UART-регистрами. Чтобы работать с ними, нам нужно создать экземпляр `SerialPort`. Для этого мы создаем новый модуль `serial` со следующим содержимым:

```rust
// в src/main.rs

mod serial;
```

```rust
// в src/serial.rs

use uart_16550::SerialPort;
use spin::Mutex;
use lazy_static::lazy_static;

lazy_static! {
    pub static ref SERIAL1: Mutex<SerialPort> = {
        let mut serial_port = unsafe { SerialPort::new(0x3F8) };
        serial_port.init();
        Mutex::new(serial_port)
    };
}
```

Like with the [VGA text buffer][vga lazy-static], we use `lazy_static` and a spinlock to create a `static` writer instance. By using `lazy_static` we can ensure that the `init` method is called exactly once on its first use.

Like the `isa-debug-exit` device, the UART is programmed using port I/O. Since the UART is more complex, it uses multiple I/O ports for programming different device registers. The unsafe `SerialPort::new` function expects the address of the first I/O port of the UART as argument, from which it can calculate the addresses of all needed ports. We're passing the port address `0x3F8`, which is the standard port number for the first serial interface.

[vga lazy-static]: @/edition-2/posts/03-vga-text-buffer/index.md#lazy-statics

To make the serial port easily usable, we add `serial_print!` and `serial_println!` macros:

```rust
// в src/serial.rs

#[doc(hidden)]
pub fn _print(args: ::core::fmt::Arguments) {
    use core::fmt::Write;
    SERIAL1.lock().write_fmt(args).expect("Printing to serial failed");
}

/// Prints to the host through the serial interface.
#[macro_export]
macro_rules! serial_print {
    ($($arg:tt)*) => {
        $crate::serial::_print(format_args!($($arg)*));
    };
}

/// Prints to the host through the serial interface, appending a newline.
#[macro_export]
macro_rules! serial_println {
    () => ($crate::serial_print!("\n"));
    ($fmt:expr) => ($crate::serial_print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => ($crate::serial_print!(
        concat!($fmt, "\n"), $($arg)*));
}
```

The implementation is very similar to the implementation of our `print` and `println` macros. Since the `SerialPort` type already implements the [`fmt::Write`] trait, we don't need to provide our own implementation.

[`fmt::Write`]: https://doc.rust-lang.org/nightly/core/fmt/trait.Write.html

Now we can print to the serial interface instead of the VGA text buffer in our test code:

```rust
// в src/main.rs

#[cfg(test)]
fn test_runner(tests: &[&dyn Fn()]) {
    serial_println!("Running {} tests", tests.len());
    […]
}

#[test_case]
fn trivial_assertion() {
    serial_print!("trivial assertion... ");
    assert_eq!(1, 1);
    serial_println!("[ok]");
}
```

Note that the `serial_println` macro lives directly under the root namespace because we used the `#[macro_export]` attribute, so importing it through `use crate::serial::serial_println` will not work.

### QEMU Arguments

To see the serial output from QEMU, we need use the `-serial` argument to redirect the output to stdout:

```toml
# в Cargo.toml

[package.metadata.bootimage]
test-args = [
    "-device", "isa-debug-exit,iobase=0xf4,iosize=0x04", "-serial", "stdio"
]
```

When we run `cargo test` now, we see the test output directly in the console:

```console
$ cargo test
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running target/x86_64-blog_os/debug/deps/blog_os-7b7c37b4ad62551a
Building bootloader
    Finished release [optimized + debuginfo] target(s) in 0.02s
Running: `qemu-system-x86_64 -drive format=raw,file=/…/target/x86_64-blog_os/debug/
    deps/bootimage-blog_os-7b7c37b4ad62551a.bin -device
    isa-debug-exit,iobase=0xf4,iosize=0x04 -serial stdio`
Running 1 tests
trivial assertion... [ok]
```

However, when a test fails we still see the output inside QEMU because our panic handler still uses `println`. To simulate this, we can change the assertion in our `trivial_assertion` test to `assert_eq!(0, 1)`:

![QEMU printing "Hello World!" and "panicked at 'assertion failed: `(left == right)`
    left: `0`, right: `1`', src/main.rs:55:5](qemu-failed-test.png)

We see that the panic message is still printed to the VGA buffer, while the other test output is printed to the serial port. The panic message is quite useful, so it would be useful to see it in the console too.

### Print an Error Message on Panic

To exit QEMU with an error message on a panic, we can use [conditional compilation] to use a different panic handler in testing mode:

[conditional compilation]: https://doc.rust-lang.org/1.30.0/book/first-edition/conditional-compilation.html

```rust
// в src/main.rs

// our existing panic handler
#[cfg(not(test))] // new attribute
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{}", info);
    loop {}
}

// our panic handler in test mode
#[cfg(test)]
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    serial_println!("[failed]\n");
    serial_println!("Error: {}\n", info);
    exit_qemu(QemuExitCode::Failed);
    loop {}
}
```

For our test panic handler, we use `serial_println` instead of `println` and then exit QEMU with a failure exit code. Note that we still need an endless `loop` after the `exit_qemu` call because the compiler does not know that the `isa-debug-exit` device causes a program exit.

Now QEMU also exits for failed tests and prints a useful error message on the console:

```console
$ cargo test
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running target/x86_64-blog_os/debug/deps/blog_os-7b7c37b4ad62551a
Building bootloader
    Finished release [optimized + debuginfo] target(s) in 0.02s
Running: `qemu-system-x86_64 -drive format=raw,file=/…/target/x86_64-blog_os/debug/
    deps/bootimage-blog_os-7b7c37b4ad62551a.bin -device
    isa-debug-exit,iobase=0xf4,iosize=0x04 -serial stdio`
Running 1 tests
trivial assertion... [failed]

Error: panicked at 'assertion failed: `(left == right)`
  left: `0`,
 right: `1`', src/main.rs:65:5
```

Since we see all test output on the console now, we no longer need the QEMU window that pops up for a short time. So we can hide it completely.

### Hiding QEMU

Since we report out the complete test results using the `isa-debug-exit` device and the serial port, we don't need the QEMU window anymore. We can hide it by passing the `-display none` argument to QEMU:

```toml
# в Cargo.toml

[package.metadata.bootimage]
test-args = [
    "-device", "isa-debug-exit,iobase=0xf4,iosize=0x04", "-serial", "stdio",
    "-display", "none"
]
```

Now QEMU runs completely in the background and no window is opened anymore. This is not only less annoying, but also allows our test framework to run in environments without a graphical user interface, such as CI services or [SSH] connections.

[SSH]: https://en.wikipedia.org/wiki/Secure_Shell

### Timeouts

Since `cargo test` waits until the test runner exits, a test that never returns can block the test runner forever. That's unfortunate, but not a big problem in practice since it's normally easy to avoid endless loops. In our case, however, endless loops can occur in various situations:

- The bootloader fails to load our kernel, which causes the system to reboot endlessly.
- The BIOS/UEFI firmware fails to load the bootloader, which causes the same endless rebooting.
- The CPU enters a `loop {}` statement at the end of some of our functions, for example because the QEMU exit device doesn't work properly.
- The hardware causes a system reset, for example when a CPU exception is not caught (explained in a future post).

Since endless loops can occur in so many situations, the `bootimage` tool sets a timeout of 5 minutes for each test executable by default. If the test does not finish in this time, it is marked as failed and a "Timed Out" error is printed to the console. This feature ensures that tests that are stuck in an endless loop don't block `cargo test` forever.

You can try it yourself by adding a `loop {}` statement in the `trivial_assertion` test. When you run `cargo test`, you see that the test is marked as timed out after 5 minutes. The timeout duration is [configurable][bootimage config] through a `test-timeout` key in the Cargo.toml:

[bootimage config]: https://github.com/rust-osdev/bootimage#configuration

```toml
# в Cargo.toml

[package.metadata.bootimage]
test-timeout = 300          # (in seconds)
```

If you don't want to wait 5 minutes for the `trivial_assertion` test to time out, you can temporarily decrease the above value.

### Insert Printing Automatically

Our `trivial_assertion` test currently needs to print its own status information using `serial_print!`/`serial_println!`:

```rust
#[test_case]
fn trivial_assertion() {
    serial_print!("trivial assertion... ");
    assert_eq!(1, 1);
    serial_println!("[ok]");
}
```

Manually adding these print statements for every test we write is cumbersome, so let's update our `test_runner` to print these messages automatically. To do that, we need to create a new `Testable` trait:

```rust
// в src/main.rs

pub trait Testable {
    fn run(&self) -> ();
}
```

The trick now is to implement this trait for all types `T` that implement the [`Fn()` trait]:

[`Fn()` trait]: https://doc.rust-lang.org/stable/core/ops/trait.Fn.html

```rust
// в src/main.rs

impl<T> Testable for T
where
    T: Fn(),
{
    fn run(&self) {
        serial_print!("{}...\t", core::any::type_name::<T>());
        self();
        serial_println!("[ok]");
    }
}
```

We implement the `run` function by first printing the function name using the [`any::type_name`] function. This function is implemented directly in the compiler and returns a string description of every type. For functions, the type is their name, so this is exactly what we want in this case. The `\t` character is the [tab character], which adds some alignment to the `[ok]` messages.

[`any::type_name`]: https://doc.rust-lang.org/stable/core/any/fn.type_name.html
[tab character]: https://en.wikipedia.org/wiki/Tab_key#Tab_characters

After printing the function name, we invoke the test function through `self()`. This only works because we require that `self` implements the `Fn()` trait. After the test function returned, we print `[ok]` to indicate that the function did not panic.

The last step is to update our `test_runner` to use the new `Testable` trait:

```rust
// в src/main.rs

#[cfg(test)]
pub fn test_runner(tests: &[&dyn Testable]) {
    serial_println!("Running {} tests", tests.len());
    for test in tests {
        test.run(); // новое
    }
    exit_qemu(QemuExitCode::Success);
}
```

The only two changes are the type of the `tests` argument from `&[&dyn Fn()]` to `&[&dyn Testable]` and that we now call `test.run()` instead of `test()`.

We can now remove the print statements from our `trivial_assertion` test since they're now printed automatically:

```rust
// в src/main.rs

#[test_case]
fn trivial_assertion() {
    assert_eq!(1, 1);
}
```

The `cargo test` output now looks like this:

```console
Running 1 tests
blog_os::trivial_assertion...   [ok]
```

The function name now includes the full path to the function, which is useful when test functions in different modules have the same name. Otherwise the output looks the same as before, but we no longer need to manually add print statements to our tests.
