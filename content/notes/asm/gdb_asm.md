---
layout: page
title: GDB commands for Assembly
date: 2021-02-08
---

Start up:
```
gdb exe
```

Change the assembler syntax from AT&T to Intel:
```
set disassembly-flavor intel
```
If you want to make this setting permanent, paste that into `~/gdbinit`.

You can verify which assembly syntax is active with:
```
show disassembly-flavor
```

Enter TUI mode:
```
Ctrl+X, Ctrl+A
```

Switch to assembly mode:
```
layout asm
```

Run the program and break in the first like of work (`b main` and `b _start` don't work):
```
starti
```

Execute till next instruction:
```
ni
```

Step one instruction (unlike the previous, this will enter function calls):
```
si
```

Retrive the values of all registers:
```
i r                      ; info registers
```

Get the value of one register:
```
i r rax
```