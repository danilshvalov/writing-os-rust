

## Процесс загрузки
Когда вы включаете ваш компьютер, он начинает выполнение кода прошивки, который хранится в постоянной памяти материнской платы. Этот код выполняет [самотестирование при включении] (power-on self-test), обнаруживает доступную ОЗУ и предварительно инициализирует CPU и другое оборудование. После этого он ищет загрузочный диск и начинает загрузку ядра операционной системы.

[ROM]: https://en.wikipedia.org/wiki/Read-only_memory
[power-on self-test]: https://en.wikipedia.org/wiki/Power-on_self-test

На архитектуре x86 существует два стандарта прошивок: "Basic Input/Output System" (**[BIOS]**) и более новый "Unified Extensible Firmware Interface" (**[UEFI]**). Стандарт BIOS считается устаревшим, но простым и хорошо поддерживается на x86 машинах с 1980-х годов. UEFI, напротив, более современный и содержит гораздо больше возможностей, а потому и более сложен в настройке (по крайней мере, на мой взгляд).

[BIOS]: https://en.wikipedia.org/wiki/BIOS
[UEFI]: https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface

В настоящее время мы предоставляем только поддержку BIOS, но также планируется поддержка UEFI. Если вы хотите помочь нам в этом, ознакомьтесь с [Github issue](https://github.com/phil-opp/blog_os/issues/349).

### Загрузка BIOS

Почти все x86 системы имеют поддержку BIOS, в том числе и новые машины на UEFI. Это здорово, потому что мы можем использовать одну и ту же логику загрузки на всех машинах, как на современных, так и на компьютерах прошлого столетия. Но у этого решения есть свой недостаток. При загрузке BIOS процессор переходит в 16-битный режим совместимости, также известный как [real mode], благодаря чему и работают все старые загрузчики, написанные еще в 1980-х. 

Но давайте начнем с самого начала:

Когда вы включаете компьютер, он загружает BIOS со специального флеш-накопителя, расположенного на материнской плате. BIOS запускает процедуры самопроверки и инициализации оборудования, а затем он ищет загрузочные диски. Если он его находит, то управление передается _загрузчику_, который представляет собой исполняемый файл, хранящийся на первых 512 байтах в начале диска. Большинство загрузчиков имеют размер более 512 байт, поэтому загрузчики обычно разделяют на две части: первую маленькую, которая умещается в 512 байт, и вторую, которая впоследствии загружается первой частью.

Загрузчик должен определить расположение образа ядра на диске и загрузить его в память. Также необходимо переключить режим работы процессора с 16-битного [real mode] сначала в 32-битный [protected mode], а затем в 64-битный [long mode], где 64-битные регистры и вся основная память будет доступна. Это его третья задача - запросить определенную информацию (например, memory map) из BIOS и передать ее ядру ОС.

[real mode]: https://en.wikipedia.org/wiki/Real_mode
[protected mode]: https://en.wikipedia.org/wiki/Protected_mode
[long mode]: https://en.wikipedia.org/wiki/Long_mode
[memory segmentation]: https://en.wikipedia.org/wiki/X86_memory_segmentation

Writing a bootloader is a bit cumbersome as it requires assembly language and a lot of non insightful steps like “write this magic value to this processor register”. Therefore we don't cover bootloader creation in this post and instead provide a tool named [bootimage] that automatically prepends a bootloader to your kernel.

Написание загрузчика - немного обременительное занятие. Необходимо знать язык ассемблера, понимать порой непонятные шаги, такие как "записать это магическое значение в регистр процессора". Поэтому мы не будем рассматривать создание загрузчика, а вместо этого предоставим эту работу [bootimage] - инструменту, который автоматически добавит загрузчик к вашему ядру ОС.

[bootimage]: https://github.com/rust-osdev/bootimage

If you are interested in building your own bootloader: Stay tuned, a set of posts on this topic is already planned! <!-- , check out our “_[Writing a Bootloader]_” posts, where we explain in detail how a bootloader is built. -->

#### Стандарт Multiboot 

Чтобы избежать таких трудностей, каждая система реализует собственный загрузчик, который совместим только с одной ОС. В 1995 году [Free Software Foundation] создал открытый стандарт загрузчиков, также известный как [Multiboot]. Стандарт определяет интерфейс между загрузчиком и ОС, поэтому любой Multiboot-совместимый загрузчик может загружать любую Multiboot-совместимую операционную систему. Эталонной реализацией принято считать [GNU GRUB] - самый популярный загрузчик для Linux систем. 

[Free Software Foundation]: https://en.wikipedia.org/wiki/Free_Software_Foundation
[Multiboot]: https://wiki.osdev.org/Multiboot
[GNU GRUB]: https://en.wikipedia.org/wiki/GNU_GRUB

Чтобы сделать Multiboot-совместимое ядро ОС, достаточно вставить [Multiboot header] в начало файла ядра. Это делает загрузку ОС в GRUB очень простым. Однако у GRUB, наряду со стандартом Multiboot, есть проблемы:

[Multiboot header]: https://www.gnu.org/software/grub/manual/multiboot/multiboot.html#OS-image-format

- Они поддерживают только 32-битный protected mode. Это значит, что необходимо выполнить настройку CPU для переключения в 64-битный long mode.
- Они предназначены для упрощения работы с загрузчиком, а не с ядром ОС. Например, ядру обязательно нужно установить [скорректированный размер страницы по умолчанию],потому что иначе GRUB не сможет найти заголовок Multiboot. Другой пример: [загрузочная информация], которая передается ядру, содержит множество архитектурно-зависимых структур вместо предоставления чистых абстракций.
- И GRUB, и Multiboot плохо документированы.
- GRUB должен быть установлен на хост-системе для создания образа загрузочного диска из файла ядра. Это затрудняет разработку на Windows или Mac.

[adjusted default page size]: https://wiki.osdev.org/Multiboot#Multiboot_2
[boot information]: https://www.gnu.org/software/grub/manual/multiboot/multiboot.html#Boot-information-format

Из-за этих недостатков мы решили не использовать GRUB и Multiboot. Однако мы планируем добавить поддержку Multiboot для [bootimage], чтобы было возможно загружать ядро ОС через GRUB. Если вы заинтересованы в создании ядра, совместимого с Multiboot, ознакомьтесь с [первым изданием] этой книги.

[first edition]: @/edition-1/_index.md

### UEFI

На данный момент мы не предоставляем поддержку UEFI, но будем рады, если вы поможете нам в этом! Пожалуйста, сообщите нам в [Github issue](https://github.com/phil-opp/blog_os/issues/349).