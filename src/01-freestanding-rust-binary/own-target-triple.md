# Создание собственного целевого триплета

**Триплет** - это стандартный термин, используемый при кросс-компиляции как способ описать целевое окружение (архитектуру процессора, поставщика, ОС, [ABI] и т. д.) под одним удобным именем. Приведем пример такого триплета в Rust, чтобы стало чуточку понятнее. Триплет с названием `x86_64-unknown-linux-gnu` описывает систему с архитектурой `x86_64`, без четкого поставщика, с операционной системой Linux и GNU ABI.

Cargo поддерживает различные триплеты. Чтобы указать cargo какой триплет использовать, необходимо указать имя триплета вместе с флагом `--target`.  Rust поддерживает [большое количество разнообразных триплетов][platform-support].

[target triple]: https://clang.llvm.org/docs/CrossCompilation.html#target-triple
[ABI]: https://stackoverflow.com/a/2456882
[platform-support]: https://forge.rust-lang.org/release/platform-support.html
[custom-targets]: https://doc.rust-lang.org/nightly/rustc/targets/custom.html

Однако для нашей буквально голой системы нам требуются указать некоторые специальные параметры конфигурации (например отсутствие базовой ОС), поэтому ни одна из [существующих целей][platform-support] нам не подходит. К счастью, Rust позволяет нам определить свою [собственную цель][custom-targets] с помощью JSON-файла. Например, JSON-файл для цели `x86_64-unknown-linux-gnu` выглядит так:

```json
{
    "llvm-target": "x86_64-unknown-linux-gnu",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "linux",
    "executables": true,
    "linker-flavor": "gcc",
    "pre-link-args": ["-m64"],
    "morestack": false
}
```

Большинство полей требуются LLVM для генерации кода для этой платформы. Например, поле [`data-layout`] определяет размер различных целочисленных типов, типов с плавающей запятой и указателей. Также есть поля, которые Rust использует для условной компиляции, например, поле `target-pointer-width`. Третий тип полей определяет, как должен быть скомпилирован крейт. Например, поле `pre-link-args` определяет аргументы, которые будут переданы [компоновщику][linker].

[`data-layout`]: https://llvm.org/docs/LangRef.html#data-layout
[linker]: https://en.wikipedia.org/wiki/Linker_(computing)

Мы будем компилировать наше ядро для `x86_64`, поэтому наше описание цели будет похоже на указанное выше. Начнем с создания файла `x86_64-blog_os.json` (можно выбрать любое имя) со следующим содержимым:

```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true
}
```

Обратите внимание, что мы изменили ОС в поле `llvm-target` и там, где раньше был `linux`, теперь стоит `none`.

Добавим еще два поля:

```json
"linker-flavor": "ld.lld",
"linker": "rust-lld",
```

Вместо использования компоновщика по умолчанию (который может не поддерживать цели Linux), мы используем кроссплатформенный компоновщик [LLD], поставляемый с Rust, для компоновки нашего ядра.

[LLD]: https://lld.llvm.org/

```json
"panic-strategy": "abort",
```

This setting specifies that the target doesn't support [stack unwinding] on panic, so instead the program should abort directly. This has the same effect as the `panic = "abort"` option in our Cargo.toml, so we can remove it from there. (Note that in contrast to the Cargo.toml option, this target option also applies when we recompile the `core` library later in this post. So be sure to add this option, even if you prefer to keep the Cargo.toml option.)

TODO перечитать
Параметр `panic-strategy` указывает, что цель не поддерживает [раскрутку стека] при панике, поэтому, вместо раскрутки, программа должна сразу завершать свою работу. Такой же эффект мы уже получали с помощью `panic = "abort"` в нашем `Cargo.toml`. Значит, что мы можем удалить `panic = "abort"` из конфигурационного файла. (Обратите внимание, что в отличие от параметра в `Cargo.toml`, `panic-strategy` применится и при компиляции `core` в этой главе.)

С помощью параметра `panic-strategy` мы указываем, что в случае паники наша программа не должна начинать раскрутку стека, а сразу должна завершиться. 

[stack unwinding]: https://www.bogotobogo.com/cplusplus/stackunwinding.php



[disabling the red zone]: @/edition-2/posts/02-minimal-rust-kernel/disable-red-zone/index.md

```json
"features": "-mmx,-sse,+soft-float",
```

The `features` field enables/disables target features. We disable the `mmx` and `sse` features by prefixing them with a minus and enable the `soft-float` feature by prefixing it with a plus. Note that there must be no spaces between different flags, otherwise LLVM fails to interpret the features string.

Поле `features` включает и отключает функции, указываемые для цели. Мы отключаем `mmx` и `sse`, добавляя к ним знак минуса и включаем `soft-float` с помощью знака плюса. Обратите внимание, что между флагами не должно быть пробелов.

`mmx` и `sse` определяют поддержку инструкций [Single Instruction Multiple Data (SIMD)], которые часто могут значительно ускорить работу программы. Однако использование больших SIMD-регистров в ядре ОС приводит к проблемам с производительностью. Причина в том, что ядру необходимо приводить все регистры в исходное состояние перед продолжением прерванной программы. Это означает, что ядро должно сохранить полное SIMD-состояние в основной памяти при каждом вызове или аппаратном прерывании. Поскольку SIMD-состояние очень велико (512-1600 байт), а прерывания могут происходить очень часто, эти доп. операции сохранения и восстановления значительно снижают производительность. Чтобы этого избежать, мы отключаем SIMD для нашего ядра (но не для приложений, работающих поверх ядра!).

[Single Instruction Multiple Data (SIMD)]: https://en.wikipedia.org/wiki/SIMD

Отключая SIMD, возникает другая проблема. По умолчанию для операций с плавающей запятой на `x86_64` требуются SIMD-регистры. Поэтому мы добавляем `soft-float` функцию, которая эмулирует все операции с плавающей запятой с помощью программных функций, основанных на обычных целых числах.

For more information, see our post on [disabling SIMD](@/edition-2/posts/02-minimal-rust-kernel/disable-simd/index.md).

Подробнее об отключении SIMD можно узнать [здесь].