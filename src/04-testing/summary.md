# Резюме

Тестирование — это очень полезная практика, которая позволяет убедиться, что определенные компоненты имеют желаемое поведение. Хоть и наличие тестов не говорит об отсутствии ошибок в программе, они по-прежнему являются полезным инструментом для их поиска, а также для предотвращения регрессий.

В этой главе мы узнали как настроить тестовый фреймворк для нашего ядра. Мы использовали свой пользовательский фреймворк для поддержки атрибута `#[test_case]`, поскольку не можем подключить стандартную библиотеку. С помощью QEMU-устройства `isa-debug-exit` наш test runner может выйти из QEMU после запуска тестов и сообщить о результатах тестирования. Чтобы вывести сообщение об ошибке в консоль вместо VGA-буфера, мы создали простой драйвер для последовательного порта.

После создания нескольких тестов для нашего макроса `println`, мы рассмотрели интеграционные тесты во второй половине этой главы. Теперь вы знаете, что такие тесты располагают в каталоге `tests`, а по своей сути они рассматриваются как полностью отдельные исполняемые файлы. Чтобы предоставить им доступ к функции `exit_qemu` и к макросу `serial_println`, мы превратили часть кода в библиотеку, которая может быть импортирована всеми исполняемыми файлами и интеграционными тестами. Поскольку интеграционные тесты выполняются в отдельной среде, они позволяют тестировать взаимодействие с оборудованием. Кроме того, с их помощью мы можем создавать тесты, которые должны вызывать панику.

Теперь у нас есть тестовый фреймворк, который работает в реалистичной среде внутри QEMU. Создавая больше тестов в следующих главах, мы сможем без особых проблем поддерживать наше ядро, когда оно станет более сложным.
