---
layout: post
title: Sorting multiple vectors in the same way using C++ variadic templates
date: 2017-09-30
published: true
---

Let's imagine we have a table like shown below. In this simple example we have information about people organized in rows.

| Id |  Name  | Surname | Age |
|:--:|:------:|---------|-----|
| 1  | Tuket  | Troco   | 24  |
| 0  |  Rufol | Conino  | 11  |
| 2  | Muscul | Man     | 20  |
| 3  | Jigoro | Kano    | 78  |

And we represent this table using C++ `std::vector`s:

```cpp
using namespace std;
...
vector<int> id = {1, 0, 2, 3};
vector<string> name = {"Tuket", "Rufol", "Muscul", "Jigoro"};
vector<string> surname = {"Troco", "Conino", "Man", "Kano"};
vector<unsigned> age = {24, 11, 20, 78};
```

We want to be able to sort these rows by different criterias. For example, we might want to sort by ascending surname alphabetical order:

| Id |  Name  | Surname | Age |
|:--:|:------:|---------|-----|
| 0  |  Rufol | Conino  | 11  |
| 3  | Jigoro | Kano    | 78  |
| 2  | Muscul | Man     | 20  |
| 1  | Tuket  | Troco   | 24  |

Our goal is to be able to do this with a very short and easy syntax:

```cpp
// the first parameter is the vector that is used as criteria (not modified)
// the rest are the vectors that are rearranged
sortVectorsAscending(surname, id, name, surname, age);
```

If we are interested we can even choose to sort only a subset of these vectors. For example, we want people names and surnames sorted by surname:

```cpp
sortVectorsAscending(surname, names, surnames);
```

We will also be able to specify the compare function.

```cpp
sortVectors(surname, greater<string>(), id, name, surname, age);    // descending
```

For this we will make use of a very convenient C++ feature called varadic templates[^variadic]. Varadic templates are just templates with a variable amount of parameters, similar to variadic functions in C.

The algorithm is very simple:
```
1. Find the sort permutation. That is, a list of indices
   pointing to the criteria vector elements in the correct order.
2. Apply permutation to all the desired vectors
```

## Get sort permutation

First we declare the function that will get the list of _sorting_ indices.

```cpp
template <typename T, typename Compare>
void getSortPermutation(
    std::vector<unsigned>& out,
    const std::vector<T>& v,
    Compare compare = std::less<T>())
{
    out.resize(v.size());
    std::iota(out.begin(), out.end(), 0);
    
    std::sort(out.begin(), out.end(),
        [&](unsigned i, unsigned j){ return compare(v[i], v[j]); });
}
```

## Apply the permutation

Next we define the function that takes the list of sorted indices and performs the permutation. This permutation could be performed _in-place_[^inplace] but it is a bit slower.

```cpp
template <typename T>
void applyPermutation(
    const std::vector<unsigned>& order,
    std::vector<T>& t)
{
    assert(order.size() == t.size());
    std::vector<T> st(t.size());
    for(unsigned i=0; i<t.size(); i++)
    {
        st[i] = t[order[i]];
    }
    t = st;
}

template <typename T, typename... S>
void applyPermutation(
    const std::vector<unsigned>& order,
    std::vector<T>& t,
    std::vector<S>&... s)
{
    applyPermutation(order, t);
    applyPermutation(order, s...);
}
```

As you can see, there are two functions with the same name. Essentially, this variadic template approach is using recusion. The fist function template is the base case, it will be checked for signature matching before the second one. The second funtion template calls itself recursivelly.

Finally we define our sort function.

```cpp
// sort multiple vectors using the criteria of the first one
template<typename T, typename Compare, typename... SS>
void sortVectors(
    const std::vector<T>& t,
    Compare comp,
    std::vector<SS>&... ss)
{
    std::vector<unsigned> order;
    getSortPermutation(order, t, comp);
    applyPermutation(order, ss...);
}
```
Variadic templates require that at least one argument is provided. So if we don't provide any vector to be sorted we get an error.

If we want to make our code less verbose, we can declare aliases for most comon compare functions.

```cpp
template<typename T, typename... SS>
void sortVectorsAscending(
    const std::vector<T>& t,
    std::vector<SS>&... ss)
{
    sortVectors(t, std::less<T>(), ss...);
}
```

Here's a simple test:

```cpp
int main()
{
  vector<int> id = {1, 0, 2, 3};
  vector<string> name = {"Tuket", "Rufol", "Muscul", "Jigoro"};
  vector<string> surname = {"Troco", "Conino", "Man", "Kano"};
  vector<unsigned> age = {24, 11, 20, 78};
  
  sortVectors(surname, greater<string>(), id, name, surname, age);
  
  for(int i=0; i<4; i++)
  {
    cout
        << setw(3) << id[i] << " "
        << setw(10) << name[i] << " "
        << setw(10) << surname[i] << " "
        << setw(3) << age[i] << endl;
  }
}
```

That's it.

Complete code in gist: [link](https://gist.github.com/tuket/df944e4194d39061b9a9f2780e0d2aae)

Test it in Ideone:
<script src="https://ideone.com/e.js/QEAoqL" type="text/javascript" ></script>

For more ways of doing this kind of sorting check out this [Stack Overflow question]([https://stackoverflow.com/questions/17074324/how-can-i-sort-two-vectors-in-the-same-way-with-criteria-that-uses-only-one-of).

# References

[^inplace]: https://blog.merovius.de/2014/08/12/applying-permutation-in-constant.html
[^variadic]: https://eli.thegreenplace.net/2014/variadic-templates-in-c/
