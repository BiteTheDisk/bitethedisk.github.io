---
title: "Rust Heap and C"
date: 2023-07-07T22:35:05+08:00
draft: true
---

Rust 和 C 交互的时候需要注意 Rust 的堆，当从 Rust 程序 fork 出一个进程
来执行 C 程序的时候，C 是不能识别 Rust 的堆的(包括其堆管理器这些)，所以当
使用 C 程序去访问 fork 后的 Rust 堆上的对象，或者传递相关堆上对象的时候可能
会产生致命问题(如非法内存访问---访问越界)