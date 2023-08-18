---
title: "Rust 和 C 的交互"
date: 2023-08-01T21:31:27+08:00
tags: ["全国赛第一阶段", "FFI"]
draft: false
---

在 Rust 中，字符串是一个胖指针，包括具有所有权的 `String` 以及 `&'a
str`，并且为 utf-8 编码

而在 C 中，字符串只是一个单纯的指针，以有符号的 `i8` 为单元，以 `\0`
作为结尾，长度不包括结尾 '\0'

在使用 Rust 编写 initrpoc 和
系统调用的时候，由于测例所要求的的接口所使用的的字符串相关参数均为 C
标准，所以我们需要将 Rust 和 C 之间的字符串进行转换

<!--more-->

## CString
Rust 中提供了
[`alloc::ffi:CString`](https://doc.rust-lang.org/alloc/ffi/struct.CString.html)
来处理两者之间的转换

### 创建 CString
一般可以通过
[`CString::new`](https://doc.rust-lang.org/alloc/ffi/struct.CString.html#method.new)
来创建一个 `CString`

```rust
pub fn new<T: Into<Vec<u8>>>(t: T) -> Result<CString, NulError>
```

具体如下:

```rust
#[macro_use]
extern crate alloc;

use alloc::ffi::CString;

fn main() {
    let c_string = CString::new("Hello, world!").unwrap();
    // ...
}
```

这里的 `new` 方法会检查传入的值是否是以 `\0`
结尾的，**该方法本身会将传入的值自动加上
`\0`，所以要求传入的参数所表示的字符串不能以 `\0` 结尾**

也可以通过 unsafe 方法
[`CString::from_vec_unchecked`](https://doc.rust-lang.org/alloc/ffi/struct.CString.html#method.from_vec_unchecked)
等忽略这种检查

### CString 转换需注意
Rust 是一门强调安全的语言，所有的值都需要在超出各自的声明周期后释放其所占有的内存

文档中提供了一个 `pub fn into_raw(self) -> *mut c_char`
方法，该方法的实际意义和它的签名并不相同: **其返回的字符串必须不可变！**

因为从 Rust 向 C 传递的字符串本质上还是当前 Rust
程序分配的内存(在堆上，本质还是一个 `Vec<u8>`，对，是 `u8`，而不是
`i8`，估计是为了防止由于溢出而导致的编译失败，rustc dddd)，如果在 C
程序中改变了当前字符串的值，比如删去结尾 `\0`，或者将 `\0` 提前，都会带来内存安全隐患

所以，这里的 `*mut c_char` 实际对应 C 中的 `char *`

### 及时回收 CString 防止内存泄露
前面提到，`CString` 底层其实是 `Vec<u8>`，其实际上是分配在 Rust 的堆上的

在将 `CString` leek 成传给 C 的指针后，`CString` 底层的 `Vec<u8>` 就脱离了 Rust
所有权机制的监管，如果不**重构裸指针的所有权**，那将会导致内存泄露

要想回收 `CString` 所使用的内存，可以按以下操作

```rust
#[macro_use]
extern crate alloc;

use alloc::ffi::CString;

fn main() {
    let initproc = CString::new("/busybox").unwrap();
    let initproc_raw = initproc.into_raw();

    // ...

    // Execute syscall, which uses C ABI.
    execve(iniproc, /* argv, envs */);

    // Release memory of `initproc_raw`.
    {
        unsafe {
            CString::from_raw(initproc_raw);
        }
    }
}
```
