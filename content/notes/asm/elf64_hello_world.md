---
layout: page
title: ELF64 hello world
date: 2021-02-10
---

I like to understand how things work under the hood. Assembly programming is very close to the metal but, even then, the assembler hides some complexity to make our lives easier. Also the linker does lots of work I don't completely understand.

So I though I would make an executable by hand, where every bit in it is understood. The resulting ELF64 executable is very small (160 bytes) compared with the executables generated by C compilers and even assemblers.

I found the [wikipedia article](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) on the ELF format to be quite handy, although I also had to look up other resources (all of them at the end).

The executable has only one program segment that contains both code and data. There are no sections. Segments and sections are different things[^so_section_vs_segment] [^section_vs_segment_ciro].

I also read [this article](https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html) about making the smallest possible ELF32 executable that returns 42. Lots of tricks are used in order to make the executable as small a possible, even some that abuse the ELF format spec, like overlapping different sections. We are not going to go as far, just try to make the simplest hello world. Also, I'm going to make an ELF64 instead. There is also [this post](https://stackoverflow.com/questions/53382589/smallest-executable-program-x86-64) in StackOverflow where they also try to make the smallest possible ELF64.

We could make our executable by typing it in a hex editor. But there are other more convenient ways.

In the resources just mentioned, they use NASM with a special flag `-f bin`. With this flag, only what is specified in the `.asm` source file will appear in the output binary file (when we compile normally, NASM creates for you the ELF headers and so on). This is what one of this `.asm` files would look like (copied from the forementioned StackOverflow post):

```nasm
bits 64
org 0x08048000

ehdr:                                     ; Elf64_Ehdr
            db  0x7F, "ELF", 2, 1, 1, 0   ;   e_ident
    times 8 db  0
            dw  2                         ;   e_type
            dw  62                        ;   e_machine
            dd  1                         ;   e_version
            dq  _start                    ;   e_entry
            dq  phdr - $$                 ;   e_phoff
            dq  0                         ;   e_shoff
            dd  0                         ;   e_flags
            dw  ehdrsize                  ;   e_ehsize
            dw  phdrsize                  ;   e_phentsize
            dw  1                         ;   e_phnum
            dw  0                         ;   e_shentsize
            dw  0                         ;   e_shnum
            dw  0                         ;   e_shstrndx

ehdrsize    equ $ - ehdr

phdr:                                     ; Elf64_Phdr
            dd  1                         ;   p_type
            dd  7                         ;   p_flags
            dq  0                         ;   p_offset
            dq  $$                        ;   p_vaddr
            dq  $$                        ;   p_paddr
            dq  filesize                  ;   p_filesz
            dq  filesize                  ;   p_memsz
            dq  0x1000                    ;   p_align

phdrsize    equ     $ - phdr

_start:
   mov di,42        ; only the low byte of the exit code is kept,
                    ; so we can use di instead of the full edi/rdi
   xor eax,eax
   mov al,60        ; shorter than mov eax,60
   syscall          ; perform the syscall

filesize      equ     $ - $$
```

And we can compile it with:

```
nasm -f bin min64.asm -o min64
```

But instead of using NASM for generating the executable, I opted to use a small C program that writes a binary file. Because in the future I want to make a compiler in C that doesn't require an assembler.

```c
#include <stdio.h>
#include <string.h>
#include <stdint.h>

typedef uint8_t u8;
typedef uint16_t u16;
typedef uint32_t u32;
typedef uint64_t u64;

typedef struct Elf64Header {
    char elfMagicNumber[4]; // "\x7fELF"
    u8 bitAmount; // 1: 32-bit, 2: 64-bit
    u8 endian; // 1: little endian, 2: big endian
    u8 elfVersion1; // must be 1
    u8 osAbi; // 0: System V, 3: Linux
    u8 abiVersion; // in statically linked executables has no effect. In dynamically linked executables, if OS_ABI==3, defines dynamic linker features
    u8 unused[7];
    u16 objFileType; // ET_EXEC=2, ET_DYN=3 (DYN is used for PIC)
    u16 arch; // 0x3E: AMD64
    u32 elfVersion2; // the elf version again, must be 1
    u64 entryPointOffset; // entry point from where the process should start executing
    u64 phtOffset; // start of the program header table
    u64 shtOffset; // start of the section header table
    u32 processorFlags; // processor-specific flags
    u16 headerSize; // the size of this header
    u16 phtEntrySize; // the size of one PHT entry
    u16 numPhtEntries; // num entries in the PHT
    u16 shtEntrySize; // the size of one SHT entry
    u16 numShtEntries; // num entries in the SHT
    u16 namesSht; // the index of the SHT entry that contains the section names
} Elf64Header;

typedef struct Elf64_PhtEntry {
    u32 segmentType; // 1: loadable segment, 2: dynamic linking info
    u32 flags; // segment-dependent flags (position for 64-bit structure)
    u64 offset; // offset of the segment in the file image
    u64 vaddr; // virtual address of the segment in memory
    u64 paddr; // on systems where the physical address is relevant, reserved for the physical address of the segment
    u64 sizeInFile; // size of the segment in the file image
    u64 sizeInMem; // size of the segment in memory
    u64 align; // 0 and 1 specify no alignment. Otherwise should be a positive, integral power of 2, with 'vaddr' equating 'offset' modulus 'p_align'
} Elf64_PhtEntry;

// https://godbolt.org/z/vh8aEe
const unsigned char asmCode[] = {
    0xb8, 0x01, 0x00, 0x00, 0x00, // mov rax, 1 (syscall: write)
    0xbf, 0x01, 0x00, 0x00, 0x00, // mov rdi, 1 (stdout)
    0x48, 0x8d, 0x35, 0x10, 0x0, 0x0, 0x0, // lea rsi, [rel helloStr]
    0xba, 0x07, 0x00, 0x00, 0x00, // mov rex, sizeof(helloStr)
    0x0f, 0x05, // syscall
    0xb8, 0x3c, 0x00, 0x00, 0x00, // mov eax, 3c
    0x48, 0x31, 0xff, // xor rdi, rdi
    0x0f, 0x05 // syscall
};
const char helloStr[] = {'h', 'e', 'l', 'l', 'o', '\n'};

int main()
{
    const int headersSize = sizeof(Elf64Header) + sizeof(Elf64_PhtEntry);
    const int fileSize = headersSize + sizeof(asmCode) + sizeof(helloStr);

    Elf64Header header = {
        .elfMagicNumber = {0x7F, 'E', 'L', 'F'},
        .bitAmount = 2, // 64-bit
        .endian = 1, // little endian
        .elfVersion1 = 1,
        .osAbi = 0, // system V
        .abiVersion = 0,
        .unused = {0,0,0,0,0,0,0},
        .objFileType = 3,
        .arch = 0x3E,
        .elfVersion2 = 1,
        .entryPointOffset = 0x400000 + headersSize,
        .phtOffset = sizeof(Elf64Header),
        .shtOffset = 0,
        .processorFlags = 0,
        .headerSize = 64,
        .phtEntrySize = sizeof(Elf64_PhtEntry),
        .numPhtEntries = 1,
        .shtEntrySize = 0,
        .numShtEntries = 0, //1,
        .namesSht = 0
    };

    Elf64_PhtEntry phtEntry = {
        .segmentType = 1, // 1: PT_LOAD
        .flags = 0x7, // 0: execute, 1: write, 2: read
        .offset = 0,
            // this one is quite weird. It looks like the entire executable need to be inside
            // some segment, including the Elf64Header
        .vaddr = 0x400000, // linux likes this address for x64
        .paddr = 0x400000,
        .sizeInFile = fileSize,
        .sizeInMem = fileSize,
        .align = 0x1000
    };

    printf("%ld\n", sizeof(Elf64Header));
    printf("%ld\n", sizeof(Elf64_PhtEntry));
    printf("%x\n", fileSize-7);

    FILE* file = fopen("raw_exe", "w");

    fwrite(&header, 1, sizeof(header), file);
    fwrite(&phtEntry, 1, sizeof(phtEntry), file);
    fwrite(asmCode, 1, sizeof(asmCode), file);
    fwrite(helloStr, 1, sizeof(helloStr), file);

    fclose(file);
}
```

Inside the [Program header](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format#Program_header), I struggled a lot with the `p_offset` field ([and I'm not the only one](https://cirosantilli.com/elf-hello-world#p-191)). If you didn't notice, the value of `p_offset` is 0. But the [spec](https://refspecs.linuxbase.org/elf/gabi4+/ch5.pheader.html) says:
> This member gives the offset from the beginning of the file at which the first byte of the segment resides

And our executable layout looks like this:

|  |  |
| :------ |:--- |
| Elf64Header | 64 bytes |
| Program Header Entry | 56 bytes |
| Program Segment | ... |

So shouldn't `p_offset` be 64+56=120?

That would make sense but there is a little problem.
The ELF spec says the following about `p_align`:
> This member holds the value to which the segments are
> aligned in memory and in the file.  Loadable process
> segments must have congruent values for p_vaddr and
> p_offset, modulo the page size.  Values of zero and one
> mean no alignment is required.  Otherwise, p_align should
> be a positive, integral power of two, and p_vaddr should
> equal p_offset, modulo p_align.

The key is in "p_vaddr should equal p_offset, modulo p_align". In our case, `p_vaddr` is `0x400000` which is the usual address for x64 processes[^linux_usual_proc_addr]. If we had `p_offset==120`, the alignment rule would be violated, since we have a `p_align` of `0x1000`.

And the obvious question is: why we don't just decrease the alignment? ... The Linux kernel has a minimum alignment that is usually the page size. I believe that the page size is usually 4KB. So we would have to add padding after the headers to align up to 4KB. That would make the executable much bigger; therefore we opted to just include the headers in the segment, and set the entry point to `0x400000 + 120`.

But there is something better we can do. We said that `0x400000` is the usual load address for x64 processes. But it turns out it's not mandatory. We can set `p_vaddr` to `0x400078` so it matches the `p_offset`, modulo `0x1000`. In this way, the executable is small and we don't load the headers unnecessarily.

In order to figure this out I had to ask in [StackOverflow](https://stackoverflow.com/questions/66035102/in-elf-why-do-the-headers-need-to-be-in-one-segment).

Here's the [final code](https://gist.github.com/tuket/6f214486487ab2592adbd4ad1778bc6b)


Other interesting resources:
- https://greek0.net/elf.html
- https://stackoverflow.com/questions/2966426/why-do-virtual-memory-addresses-for-linux-binaries-start-at-0x8048000
- https://www.intezer.com/blog/research/executable-linkable-format-101-part1-sections-segments/
- http://michalmalik.github.io/elf-dynamic-segment-struggles
- http://www.048227799.com/upload/Files/Rise%20&%20Fall%20of%20Binaries%20-%20Part%2003.pdf
- https://jeffjerseycow.github.io/2017/12/what-are-the-got-and-plt-pt1
- https://docs.oracle.com/cd/E19683-01/816-1386/6m7qcoblk/index.html#chapter6-tbl-39

[^so_section_vs_segment]: https://stackoverflow.com/questions/14361248/whats-the-difference-of-section-and-segment-in-elf-file-format
[^section_vs_segment_ciro]: https://cirosantilli.com/elf-hello-world#section-vs-segment
[^linux_usual_proc_addr]: https://stackoverflow.com/questions/14314021/why-linux-gnu-linker-chose-address-0x400000