# Тестирование

This post explores unit and integration testing in `no_std` executables. We will use Rust's support for custom test frameworks to execute test functions inside our kernel. To report the results out of QEMU, we will use different features of QEMU and the `bootimage` tool.

## Тестирование в Rust

В Rust есть [встроенная среда тестирования][built-in-tests], которая позволяет запускать юнит-тесты без необходимости что-либо настраивать. Просто создайте функцию, которая проверяет результат работы вашей программы с помощью ассертов и добавьте атрибут `#[test]` перед функцией. После этого команда `cargo test` автоматически найдет и выполнит тестирование вашего кода.

К сожалению, в нашем случае все немного сложнее. Дело в том, что фреймворк для тестирования в Rust неявно использует библиотеку [`test`][test-lib], которая зависит от стандартной библиотеки. Не подключая стандартную библиотеку, использовать фреймворк для тестирования, встроенный в Rust, мы не можем.

Это легко проверить, запустив команду `cargo test`:

```console
$ cargo test
   Compiling blog_os v0.1.0 (/…/blog_os)
error[E0463]: can't find crate for `test`
```

[built-in-tests]: https://doc.rust-lang.org/book/ch11-00-testing.html
[test-lib]: https://doc.rust-lang.org/test/index.html
