---
layout: page
title: Print the contents of a file in Linux
---

```nasm
; nasm -f elf64 pf.asm && ld -nostartfiles pf.o -o pf

SECTION .text    

global _start
_start:
    mov rax, 2 ; syscall: open
    mov rdi, str_testFile
    mov rsi, 0 ; flags
    syscall

    mov rdi, rax ; the file descriptor
    mov rsi, buffer
    mov rdx, 128
    mov rax, 0; syscall: read
    syscall

    mov rdi, 1 ; stdout
    mov rsi, buffer
    mov rdx, rax ; the size returned by read
    mov rax, 1; syscall: write
    syscall

    mov rax, 60 ; syscal: exit
    mov rdi, 0 ; return code
    syscall

str_testFile: db "test.txt", 0

SECTION .data
buffer: TIMES 128 db 0
```