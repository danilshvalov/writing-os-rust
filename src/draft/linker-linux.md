# Исправляем ошибки компоновщика на Linux

На Linux возникает следующая ошибка компоновщика (сокращенно):

```console
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

По умолчанию компоновщик запускает среду выполнения Си, которая требует некоторые символы из библиотеки `libc`, которую мы больше не подключаем из-за аттрибута `no_std`. Компоновщик не находит эти символы и выдает ошибку. Чтобы решить эту проблему, мы укажем компоновщику не запускать среду выполнения Си с помощью флага `-nostartfiles`:

```console
$ cargo rustc -- -C link-arg=-nostartfiles
```

`cargo rustc` ведет себя точно также, как и `cargo build`, но позволяет передавать параметры компилятору и компоновщику Rust.

Нам не нужно указывать имя точки входа явно, поскольку компоновщик по умолчанию ищет функцию с именем `_start`.

Теперь наша программа должна успешно собираться на Linux.
