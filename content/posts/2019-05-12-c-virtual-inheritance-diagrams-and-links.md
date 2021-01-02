---
layout: post
title: C++ virtual inheritance diagrams and links
date: 2019-05-12
published: true
---

These is a cheatsheet of the memory layout of some common pattens in virtual inheritance.

For each case there is first the code.

Then there is a diagram.  
On the left there is a representation of the inheritance relationships.
<span style="color:#A59200">Gold</span> color means virtual.  
On the right there is the memory layout.  
The dashed arrows represent where a pointer of that type would point if a casting was performed.

At the bottom of this page there is a collection of [links](#links). Some of them are good for a first contact, and other go very in depth.

# 1

```cpp
struct Widget
{
    int widget;
};

struct Button
    : public Widget
{
    int button;
}
```

<img src="/img/virtual_inheritance/1.svg" alt="1.svg" width="75%"/>

# 2

```cpp
struct Widget
{
    int widget;
    virtual void f(){}
};

struct Button
    : public Widget
{
    int button;
}
```
<img src="/img/virtual_inheritance/2.svg" alt="2.svg" width="75%"/>

# 3

```cpp
struct Widget
{
    int widget;
};

struct Button
    : public virtual Widget
{
    int button;
}
```
<img src="/img/virtual_inheritance/3.svg" alt="3.svg" width="75%"/>

# 4

```cpp
struct Widget
{
    int widget;
    virtual void f(){}
};

struct Button
    : public virtual Widget
{
    int button;
}
```
<img src="/img/virtual_inheritance/4.svg" alt="4.svg" width="75%"/>

# 5

```cpp
struct Widget
{
    int widget;
};

struct Prop1 { int prop1; };
struct Prop2 { int prop2; };
struct Prop3 { int prop3; };

struct Button
    : public Widget
    , public Prop1
    , public Prop2
    , public Prop3
{
    int button;
};
```

<img src="/img/virtual_inheritance/5.svg" alt="5.svg" width="100%"/>

# 6

```cpp
struct Widget
{
    int widget;
};

struct Prop1
    : public virtual Widget
{
    int prop1;
};

struct Prop2
    : public virtual Widget
{
    int prop2;
};

struct Prop3
    : public virtual Widget
{
    int prop3;
};

struct Button
    : public Prop1
    , public Prop2
    , public Prop3
{
    int button;
};
```

<img src="/img/virtual_inheritance/6.svg" alt="6.svg" width="100%"/>

# 7

```cpp
struct Widget
{
    int widget;
};

struct Prop1
    : public virtual Widget
{
    int prop1;
};

struct Prop2
    : public virtual Widget
{
    int prop2;
};

struct Prop3
    : public virtual Widget
{
    int prop3;
};

struct Button
    : public virtual Widget
    , public Prop1
    , public Prop2
    , public Prop3
{
    int button;
};
```

Same memory layout.

<img src="/img/virtual_inheritance/7.svg" alt="7.svg" width="100%"/>

# 8

```cpp
struct Widget
{
    int widget;
    virtual void f(){}
};

struct Prop1
    : public virtual Widget
{
    int prop1;
};

struct Prop2
    : public virtual Widget
{
    int prop2;
};

struct Prop3
    : public virtual Widget
{
    int prop3;
};
    
struct Button
    : public virtual Widget
    : public Prop1
    , public Prop2
    , public Prop3
{
    int button;
};
```

<img src="/img/virtual_inheritance/8.svg" alt="8.svg" width="100%"/>

# Links

Good as introduction:

* https://www.cprogramming.com/tutorial/virtual_inheritance.html

This series explores virtual inheritance at the low level. Goes really in depth: memory layout and assemble code. This is the best resource I found. It also teaches you how to use GDB to get the answers yourself.

* https://shaharmike.com/cpp/vtable-part1/

* https://shaharmike.com/cpp/vtable-part2/

* https://shaharmike.com/cpp/vtable-part3/

* https://shaharmike.com/cpp/vtable-part4/

Other useful resources:

* https://cs.nyu.edu/courses/fall16/CSCI-UA.0470-001/slides/MemoryLayoutMultipleInheritance.pdf

* http://www.drdobbs.com/cpp/multiple-inheritance-considered-useful/184402074?pgno=1

* https://www.programering.com/a/MTO5UTNwATE.html

* https://hownot2code.com/2016/08/12/good-and-bad-sides-of-virtual-inheritance-in-c/

Some interesting stackoverflow questions:

* https://stackoverflow.com/questions/11603198/virtual-tables-and-memory-layout-in-multiple-virtual-inheritance
* https://stackoverflow.com/questions/28521242/memory-layout-of-a-class-under-multiple-or-virtual-inheritance-and-the-vtables