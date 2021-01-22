---
layout: page
title: Assembly
published: true
---

Unless otherwise specified I use the Intel syntax. My assembler of choice is NASM but sometimes I might also use yasm, which is quite similar.

## Hello world in Linux

This is a minimal hello world program in Linux. It doesn't need to link any external libraries.

```nasm
SECTION .text

global _start ; "global" means that the symbol can be accessed in other modules. In order to refer to a global symbol from another module, you must use the "extern" keyboard
_start:
    mov eax, 4 ; syscall: write
    mov ebx, 1 ; stdout
    mov ecx, msg
    mov edx, msgLen
    syscall

    mov eax, 1 ; syscal: exit
    mov ebx, 0 ; return code
    syscall

msg: db "hello",10
msgLen: equ $ - msg ; the $ sign means the current byte address. That means the address where the next byte would go
```

I compile it with

```bash
$ nasm -f elf64 hello.asm && ld -nostartfiles hello.o -o hello
$ ./hello
hello
```

Since this program doesn't use any external functions, the executable is very small, and it doesn't link any dynamic libraries (not even libc).

```
$ ldd hello
    not a dynamic executable
$ wc -c hello
792 hello
```
So it says that the executable is 792 bytes but we can easily make it smaller by passing the `-s` flag to `ld`. Or we could also use the `strip` command.

```
$ nasm -f elf64 hello.asm && ld -s -nostartfiles hello.o -o hello
$ wc -c hello
384 hello

$ nasm -f elf64 hello.asm && ld -nostartfiles hello.o -o hello
$ strip hello
$ wc -c hello
384 hello
```

For more info about making small executable read [this article](https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html) (although it's a bit old and uses 32-bit instructions).