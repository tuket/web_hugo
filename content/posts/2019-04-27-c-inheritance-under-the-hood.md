---
layout: post
published: false
title: C++ inheritance under the hood
date: 27/4/2019
---
In this post we are going to explore how C++ inheritance works.

We will cover things like the memory layout of the objects, vtables, different types of casting and virtual inheritance.

# Simple inheritance

Let's start with the simplest kind of inheritance: only one parent, no virtual functions, no virtual inheritance.

<img src="{{site.baseurl}}/img/simple.svg" alt="simple.svg" width="100"/>

This would be the code.

```cpp
struct Widget
{
    int widget;
};

struct Button
    : public Widget
{
    int button;
};
```
[Try code](https://ideone.com/2Wj0uo)

And this would be the memory layout of an instance of `Button`

<table>
    <tr>
        <th>bytes</th>
        <th>description</th>
    </tr>
    <tr>
        <td>4</td>
        <td>widget</td>
    </tr>
    <tr>
        <td>4</td>
        <td>button</td>
    </tr>
</table>

As we can see, the memory layout of `Button` objects is just an aggregation of the member variables.

In this case, inheritance is not very different from composition:

<table>
    <tr>
        <th>Button</th>
    </tr>
    <tr>
        <td>
        <table>
            <tr>
                <th>Widget</th>
            </tr>
            <tr>
                <td>widget</td>
            </tr>
        </table>
        </td>
    </tr>
    <tr>
        <td>button</td>
    </tr>
</table>

### static_cast

In this case, `static_cast` does nothing.

We can test it with the following code.

```cpp
Button* b = new Button;
cout << b << endl;
Widget* w = static_cast<Widget*>(b);
cout << w << endl;
```
[Try code](https://ideone.com/r8epPG)

In my PC the output is:

```
0x555555767e70
0x555555767e70
```

That makes sense since, in the table we have seen before, `Widget` was at the begining of `Button`.

### dynamic_cast

`dynamic_cast` is for casting downwards.

```cpp
Widget* w = new Widget;
Button* b = dynamic_cast<Button*>(w);
```

But in this case it's not possible because `Widget` is not polymorphic. So it fails to compile.

```
error: cannot dynamic_cast ‘w’ (of type ‘struct Widget*’) to type ‘struct Button*’ (source type is not polymorphic)
```
[Try code](https://ideone.com/v2HCYO)

This is because there are no virtual functions in `Widget`. We will explore polymorphic classes later.

# Multiple inheritance

One class derives from multiple classes:

<img src="{{site.baseurl}}/img/multi.svg" alt="multi.svg" width="32%"/>

```
struct A { int a; };
struct B { int b; };
struct C { int c; };
 
struct ABC
    : public A
    , public B
    , public C
{
    int abc;
};
```
[Try code](https://ideone.com/3KRjJq)

The memory layout of ABC would come out as:

<table>
    <tr>
        <th>bytes</th>
        <th>description</th>
    </tr>
    <tr>
        <td>4</td>
        <td>a</td>
    </tr>
    <tr>
        <td>4</td>
        <td>b</td>
    </tr>
    <tr>
        <td>4</td>
        <td>c</td>
    </tr>
    <tr>
        <td>4</td>
        <td>abc</td>
    </tr>
</table>

Again, like in the the case of simple inheritance, this works just like composition.

<table>
    <tr>
        <th>ABC</th>
    </tr>
    <tr>
        <td>
        <table>
            <tr>
                <th>A</th>
            </tr>
            <tr>
                <td>a</td>
            </tr>
        </table>
        </td>
    </tr>
    <tr>
        <td>
        <table>
            <tr>
                <th>B</th>
            </tr>
            <tr>
                <td>b</td>
            </tr>
        </table>
        </td>
    </tr>
    <tr>
        <td>
        <table>
            <tr>
                <th>C</th>
            </tr>
            <tr>
                <td>c</td>
            </tr>
        </table>
        </td>
    </tr>
    <tr>
        <td>abc</td>
    </tr>
</table>

### static_cast

For multiple inheritance, `static_cast` gets a bit more interesting.

Lets run this code:
```cpp
ABC* abc = new ABC;
cout << "abc: " << abc << endl;
A* a = static_cast<A*>(abc);
cout << "a: " << a << endl;
B* b = static_cast<B*>(abc);
cout << "b: " << b << endl;
C* c = static_cast<C*>(abc);
cout << "c: " << c << endl;
```
[(Test code)](https://ideone.com/k7hkWc)

The output is:

```
abc: 0x56242a01ac20
a: 0x56242a01ac20
b: 0x56242a01ac24
c: 0x56242a01ac28
```

They are different now. That means that, in order to perform the cast, we need to increment the pointer. The offset is known at compile time.

If you like assembly, have a look at the generated machine code:
[(compiler explorer)](https://godbolt.org/z/bl-Ckj)

You will see that, for the first cast, it's not incrementing anything. However, for the other two casts, it's adding `4` and `8`.
