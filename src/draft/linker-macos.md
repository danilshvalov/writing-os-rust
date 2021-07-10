# Исправляем ошибки компоновщика на macOS

На macOS возникает следующая ошибка компоновщика (сокращенно):

```console
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: entry point (_main) undefined. for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

Компоновщик не смог функцию `_main`, которая по умолчанию является точкой входа в программу. Чтобы указать компоновщику, что функция `_start` является точкой входа, нам нужно передать флаг `-e`:

```console
$ cargo rustc -- -C link-args="-e __start"
```

`cargo rustc` ведет себя точно также, как и `cargo build`, но позволяет передавать параметры компилятору и компоновщику Rust.

Флаг `-e` указывает имя функции точки входа. Все функции в macOS должны начинаться с `_`, поэтому мы передаем имя `__start` вместо `_start`.

После этого у вас скорее всего возникнет следующая ошибка:

```console
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: dynamic main executables must link with libSystem.dylib
          for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

macOS [не поддерживает статически связанные бинарные файлы][macos-static] и требует, чтобы программы по умолчанию подключали библиотеку `libSystem`. Чтобы изменить это, мы передаем компоновщику флаг `-static`:

```console
$ cargo rustc -- -C link-args="-e __start -static"
```

Этого все еще недостаточно, поскольку возникает третья ошибка компоновщика:

```console
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: library not found for -lcrt0.o
          clang: error: linker command failed with exit code 1 […]
```

macOS по умолчанию подключает `crt0` (C runtime zero). Чтобы указать компоновщику не подключать `crt0`, мы передаем флаг `-nostartfiles`.

```console
$ cargo rustc -- -C link-args="-e __start -static -nostartfiles"
```

Теперь наша программа должна успешно собираться на macOS.

[macos-static]: https://developer.apple.com/library/archive/qa/qa1118/_index.html
