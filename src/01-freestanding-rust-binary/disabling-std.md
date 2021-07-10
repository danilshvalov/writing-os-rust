
# Отключение стандартной библиотеки

По умолчанию все крейты в Rust подключают [стандартную библиотеку][std], которая зависит от операционной системы. Стандартная библиотека использует такие ОС-зависимые возможности, как потоки, файлы, сеть. Кроме того, это в том числе и зависимость от стандартной библиотеки Си, которая взаимодействует со службами ОС. Поскольку мы планируем написать собственную ОС, нам придется отказаться от подключения стандартной библиотеки.

## Атрибут `no_std`

Сейчас наш крейт неявно подключает стандартную библиотеку. Давайте изменим это, добавив [атрибут `no_std`]:

[атрибут `no_std`]: https://doc.rust-lang.org/1.30.0/book/first-edition/using-rust-without-the-standard-library.html

```rust
// main.rs

#![no_std]

fn main() {
    println!("Hello, world!");
}
```

Если попробовать скомпилировать наш крейт (с помощью `cargo build`), то мы получим сразу несколько ошибок. Самая первая из них:

```console
error: cannot find macro `println!` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^
```

Макрос [`println`][println] является частью стандартной библиотеки, которую мы больше не подключаем. `println` использует [стандартный вывод][stdout], который представляет собой специальный файловый дескриптор, предоставляемый ОС, поэтому теперь мы не можем ничего распечатать.

Нам придется удалить `println` и оставить пустой функцию `main`, чтобы избавиться от ошибки:

```rust
// main.rs

#![no_std]

fn main() {}
```

Если снова попробовать собрать наш крейт, то мы получим следующую ошибку:

```console
$ cargo build
error: `#[panic_handler]` function required, but not found
error: language item required, but not found: `eh_personality`
```

Компилятор не смог найти функцию, помеченную атрибутом `#[panic_handler]`, а также _языковой элемент_ `eh_personality`.

[std]: https://doc.rust-lang.org/std/
[println]: https://doc.rust-lang.org/std/macro.println.html
[semver]: https://semver.org/
[stdout]: https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29
