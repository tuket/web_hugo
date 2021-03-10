---
layout: page
title: GDB commands for Assembly
date: 2021-02-08
---

Start up:
```
gdb exe
```

Enter TUI mode:
```
Ctrl+X, Ctrl+A
```

Switch to assembly mode:
```
layout asm
```

Run the program and break in the first like of work (`b main` and `b _start` dont' work):
```
starti
```

Execute till next instruction:
```
ni
```

Step one instruction (unline the previous, this will enter function calls):
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