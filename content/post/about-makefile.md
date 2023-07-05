---
title: "[区域赛] 有关 BTD 的 Makefile"
date: 2023-06-30T18:37:11+08:00
tags: ["Makefile", "Debug", "区域赛"]
draft: false
---

项目提供了一系列的 Makefile 来简化开发流程.

<!--more-->

一般只会用到项目根目录中的 Makefile:

```shell
BOOTLOADER_ELF = ./kernel/bootloader/rustsbi-qemu
KERNEL_ELF = ./kernel/target/riscv64gc-unknown-none-elf/release/kernel

sbi-qemu:
	@cp $(BOOTLOADER_ELF) sbi-qemu

kernel-qemu:
	@mv kernel/cargo kernel/.cargo
	@cd kernel/ && make kernel
	@cp $(KERNEL_ELF) kernel-qemu

all: sbi-qemu kernel-qemu

clean:
	@rm -f kernel-qemu
	@rm -f sbi-qemu
	@rm -rf build/
	@rm -rf temp/
	@cd kernel/ && cargo clean
	@cd workspace/ && make clean
	@cd fat32/ && cargo clean
	@cd misc/ && make clean

fat32img:
	@cd kernel/ && make fat32img

run:
	@cd kernel/ && make run

debug-server:
	@cd kernel/ && make debug-server

debug:
	@cd kernel/ && make debug
```

这里只需关注以下几点:

1. `run`: 构建内核并运行，内核是以 release 方式构建的
2. `debug-server`: 以 debug 方式构建内核并运行 gdb debug server
3. `debug`: 链接上面运行的 debug server 开始调试
