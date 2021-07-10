# Исправляем ошибки компоновщика на Windows

На Windows возникает следующая ошибка компоновщика (сокращенно):

```console
error: linking with `link.exe` failed: exit code: 1561
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1561: entry point must be defined
```

Ошибка `entry point must be defined` говорит нам о том, что компоновщик не смог найти точку входа. По умолчанию на Windows имя точки входа [зависит от используемой подсистемы][windows-subsystems]. Для подсистемы `CONSOLE` компоновщик ищет функцию с именем `mainCRTStartup`, для подсистемы `WINDOWS` — функцию с именем `WinMainCRTStartup`. Чтобы компоновщик искал функцию с именем `_start`, нам нужно передать флаг `/ENTRY`:

```console
$ cargo rustc -- -C link-arg=/ENTRY:_start
```

`cargo rustc` ведет себя точно также, как и `cargo build`, но позволяет передавать параметры компилятору и компоновщику Rust.

После этого у вас скорее всего возникнет следующая ошибка:

```console
error: linking with `link.exe` failed: exit code: 1221
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1221: a subsystem can't be inferred and must be
          defined
```

Исполняемые файлы Windows могут использовать разные [подсистемы][windows-subsystems]. Обычно тип подсистемы зависит от названия точки входа в программу. Если `main` является функцией точки входа, то используется подсистема `CONSOLE`, а если точка входа имеет имя `WinMain` — подсистема `WINDOWS`. Поскольку наша функция `_start` не удовлетворяет ни одному из этих требований, нам нужно явно указать тип подсистемы:

```console
$ cargo rustc -- -C link-args="/ENTRY:_start /SUBSYSTEM:console"
```

Мы используем подсистему `CONSOLE`, но если использовать подсистему `WINDOWS`, то все также будет работать. Чтобы несколько раз не записывать `-C link-arg`, мы используем флаг `-C link-args`, записывая аргументы через пробел.

Теперь наша программа должна успешно собираться на Windows.

[windows-subsystems]: https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol
