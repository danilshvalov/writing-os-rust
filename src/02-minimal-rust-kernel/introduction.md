+++
title = "A Minimal Rust Kernel"
weight = 2
path = "minimal-rust-kernel"
date = 2018-02-10

[extra]
chapter = "Bare Bones"
+++

В этой статье мы создадим небольшое 64-битное ядро на Rust для x86 архитектуры. Мы используем наработки из [предыдущей статьи], чтобы создать образ загрузочного диска, который выводит что-нибудь на экран.

TODO добавить ссылку на пред статью

<!-- more -->

This blog is openly developed on [GitHub]. If you have any problems or questions, please open an issue there. You can also leave comments [at the bottom]. The complete source code for this post can be found in the [`post-02`][post branch] branch.

[GitHub]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments
[post branch]: https://github.com/phil-opp/blog_os/tree/post-02

<!-- toc -->