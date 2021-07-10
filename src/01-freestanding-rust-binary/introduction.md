# Введение

Чтобы написать ядро операционной системы, нам нужен код, который не зависит от каких-либо функций операционной системы. Это означает, что мы не можем использовать потоки, файлы, память в куче, сеть, случайные числа, стандартный вывод или любые другие абстракции операционной системы. Все таки мы пытаемся написать собственную операционную систему.

Из-за этого мы не можем использовать большую часть [стандартной библиотеки Rust][std], но есть много возможностей из Rust, которые мы _можем_ использовать. Например, мы можем использовать [итераторы][iterators], [замыкания][closure], [pattern matching], [option] и [result], [форматирование строк][str format] и, конечно же, [систему владения][ownership]. Эти возможности позволяют написать ядро очень выразительным и высокоуровневым способом, не беспокоясь о неопределенном поведении или о небезопасном использовании памяти.

Для разработки ядра ОС на Rust нам необходимо создать исполняемый файл, который можно запускать без операционной системы. Далее мы будем называть такой файл _автономным_.

[option]: https://doc.rust-lang.org/core/option/
[result]:https://doc.rust-lang.org/core/result/
[std]: https://doc.rust-lang.org/std/
[iterators]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[closure]: https://doc.rust-lang.org/book/ch13-01-closures.html
[pattern matching]: https://doc.rust-lang.org/book/ch06-00-enums.html
[str format]: https://doc.rust-lang.org/core/macro.write.html
[ownership]: https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html
[неопределенном поведении]: https://www.nayuki.io/page/undefined-behavior-in-c-and-cplusplus-programs
[безопасности памяти]: https://tonyarcieri.com/it-s-time-for-a-memory-safety-intervention
