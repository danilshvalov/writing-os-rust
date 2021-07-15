# Пользовательские фреймворки для тестирования

К счастью, Rust позволяет заменить тестовый фреймворк по умолчанию с помощью нестабильной функции [`custom_test_frameworks`]. Эта функция не требует внешних библиотек, а значит работает без стандартной библиотеки. Она собирает все функции, которые помечены атрибутом `#[test_case]`, после чего вызывает пользовательскую функцию, передавая ей список найденных тестов. Благодаря этому у нас имеется полный контроль над процессом тестирования.

[`custom_test_frameworks`]: https://doc.rust-lang.org/unstable-book/language-features/custom-test-frameworks.html

Но у этого способа есть недостатки, например, отсутствует возможность пометить функцию как [`should_panic`][should-panic]. Вместо этого мы должны предоставить свои реализации, если они потребуются. Это идеально для нас, поскольку мы работаем с особенной средой исполнения, где обычные реализации подобных функции, вероятно, не будут работать. Например, тот же атрибут `should_panic` полагается на раскрутку стека, чтобы перехватить панику.

[should-panic]: https://doc.rust-lang.org/book/ch11-01-writing-tests.html#checking-for-panics-with-should_panic

Чтобы реализовать пользовательский фреймворк для тестирования нашего ядра, мы добавим следующее в `main.rs`:

```rust
// в src/main.rs

#![feature(custom_test_frameworks)]
#![test_runner(crate::test_runner)]

#[cfg(test)]
fn test_runner(tests: &[&dyn Fn()]) {
    println!("Running {} tests", tests.len());
    for test in tests {
        test();
    }
}
```

Наш test runner просто печатает небольшое отладочное сообщение, а затем вызывает каждую тестирующую функцию из списка. Аргумент функции `&[&dyn Fn()]` — это [_срез_][slices] ссылок [Fn()]. По сути, это список ссылок на типы, которые могут быть вызваны как функции. Поскольку функция `test_runner` бесполезна во время обычной работы программы, мы помещаем ее атрибутом  `#[cfg(test)]`, чтобы она включалась только при тестировании.

[slices]: https://doc.rust-lang.ru/book/ch04-03-slices.html
[Fn()]: https://doc.rust-lang.org/std/ops/trait.Fn.html

When we run `cargo test` now, we see that it now succeeds (if it doesn't, see the note below). However, we still see our "Hello World" instead of the message from our `test_runner`. The reason is that our `_start` function is still used as entry point. The custom test frameworks feature generates a `main` function that calls `test_runner`, but this function is ignored because we use the `#[no_main]` attribute and provide our own entry point.

Попробуйте теперь выполнить команду `cargo test`. Если ошибок не появилось — **пропустите этот абзац**. В настоящее время в cargo существует баг, который в некоторых случаях приводит к ошибке дублирования языковых элементов. Ошибка возникает, если вы установили `panic = "abort"` в конфигурационном файле `Cargo.toml`. Попробуйте убрать эту строчку, и все должно заработать. Если это не так, посмотрите методы решения проблемы [здесь](https://github.com/rust-lang/cargo/issues/7359).

Ошибок теперь нет, но мы по-прежнему видим «Hello World» вместо нашего сообщения из `test_runner`. Дело в том, что функция `_start` все еще используется как точка входа. По умолчанию `custom_test_frameworks` генерирует функцию `main`, которая и вызывает `test_runner`. Но мы то в начале файла мы указали `#[no_main]` атрибут и предоставили свою собственную точку входа.

Давайте изменим имя генерируемой функции на отличное от `main` с помощью атрибута `reexport_test_harness_main`, а затем вызовем функцию по новому имени из функции `_start`:

```rust
// в src/main.rs

#![reexport_test_harness_main = "test_main"]

#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!");

    #[cfg(test)]
    test_main();

    loop {}
}
```

Мы установили имя функции `test_main`. Теперь мы вызываем ее из нашей точки входа `_start`. Применив атрибут `#[cfg(test)]`, мы воспользовались [условной компиляцией][conditional-compilation], чтобы функция не запускалась при обычной работе программы.

Выполнив команду `cargo test`, вы увидите надпись «Running 0 tests» вместе с нашим сообщением из `test_runner`. Теперь мы готовы создать нашу первую тестирующую функцию:

```rust
// в src/main.rs

#[test_case]
fn trivial_assertion() {
    print!("trivial assertion... ");
    assert_eq!(1, 1);
    println!("[ok]");
}
```

Запустите `cargo test`. Вы должны получить:

![QEMU printing "Hello World!", "Running 1 tests", and "trivial assertion... [ok]"](qemu-test-runner-output.png)

Срез `test`, переданный нашей функции `test_runner`, теперь содержит ссылку на функцию `trivial_assertion`. Из сообщения `trivial assertion... [ok]` становится ясно, что тест отработал успешно.

После выполнения тестов наш `test_runner` возвращает управление функции `test_main`, которая возвращается в нашу точку входа `_start`. В конце функции `_start` мы попадаем в бесконечный цикл, поскольку точка входа никогда не должна что-либо возвращать. Это проблема, поскольку при вызове `cargo test` мы хотим завершить работу после выполнения всех тестов.

[conditional-compilation]: https://doc.rust-lang.org/reference/conditional-compilation.html
