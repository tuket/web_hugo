---
layout: page
title: Print the contents of a file in Linux
date: 2021-01-23
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

But that will only print 128 bytes.

If we want to print the entire file:

```nasm
; nasm -f elf64 pf.asm && ld -nostartfiles pf.o -o pf

SECTION .text    

global _start
_start:
    mov rax, 2 ; syscall: open
    mov rdi, str_testFile
    mov rsi, 0 ; flags
    syscall

    mov r10, rax ; save file descriptor
    .keep_reading:
        mov rdi, r10 ; the file descriptor
        mov rsi, buffer
        mov rdx, 128
        mov rax, 0; syscall: read
        syscall

        cmp rax, 0 ; if no bytes read, done
        jle .done_reading

        mov rdi, 1 ; stdout
        mov rsi, buffer
        mov rdx, rax ; the size returned by read
        mov rax, 1; syscall: write
        syscall

        jmp .keep_reading
    .done_reading:

    mov rax, 60 ; syscal: exit
    mov rdi, 0 ; return code
    syscall

str_testFile: db "test.txt", 0

SECTION .data
buffer: TIMES 128 db 0

```