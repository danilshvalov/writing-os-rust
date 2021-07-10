# Объединение команд сборки

Теперь у нас есть разные команды для сборки, зависящие от основной платформы, что совсем не удобно. Но это можно исправить! Создадим файл `.cargo/config.toml`, в котором укажем команды для каждой ОС:

```toml
# in .cargo/config.toml

[target.'cfg(target_os = "linux")']
rustflags = ["-C", "link-arg=-nostartfiles"]

[target.'cfg(target_os = "windows")']
rustflags = ["-C", "link-args=/ENTRY:_start /SUBSYSTEM:console"]

[target.'cfg(target_os = "macos")']
rustflags = ["-C", "link-args=-e __start -static -nostartfiles"]
```

Поле `rustflags` содержит аргументы, которые автоматически добавляются при каждом вызове `rustc`.

Больше информации о `.cargo/config.toml` можно найти в [документации][cargo-config].

Теперь наша программа должна компилироваться на всех трех платформах с помощью обычного `cargo build`.

[cargo-config]: https://doc.rust-lang.org/cargo/reference/config.html
