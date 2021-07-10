# Резюме

Небольшой автономный крейт выглядит как-то так:

`src/main.rs`:

```rust
#![no_std] // больше не подключаем стандартную библиотеку Rust
#![no_main] // отключаем точку входа, которую Rust использует по умолчанию

use core::panic::PanicInfo;

#[no_mangle] // отключаем искажение имени функции
pub extern "C" fn _start() -> ! {
    // эта функция будет точкой входа, поскольку по умолчанию 
    // компоновщик будет искать функцию с именем `_start`
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

# профиль, используемый при `cargo build`
[profile.dev]
panic = "abort" # отключаем раскрутку стека при панике

# профиль, используемый при `cargo build --release`
[profile.release]
panic = "abort" # отключаем раскрутку стека при панике
```

Чтобы скомпилировать наш крейт действительно автономным, нам нужно дополнительно передать флаг `--target thumbv7em-none-eabihf`:

```console
$ cargo build --target thumbv7em-none-eabihf
```
