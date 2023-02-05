---
layout: post
title:  "Making Sense of SFINAE"
date:   2023-02-03
categories: C++
---

SFINAE is one of those terrible acronyms, like RAII, that could only have come from C++.
And like RAII, it does neither a good job of telling you what it _does_, nor of telling
you what it's _for_. So let's make sense of Substitution Failure Is Not An Error.

## An Overload-Resolution Overview
First, some preliminaries: C++ allows you to overload functions based on the number and
types of the function's arguments. However, it also allows you to create template
functions that are generic over types, which you can also overload. When a call
expression[^1] is encountered, the compiler must pick exactly one of the overloads for
that function name to be _the_ overload that's called. The full set of rules that govern
this process are not worth enumerating here, but they tend to be guided by one major
principle: go from "more specific" to "less specific."

[^1]: Something that looks like `my_func(x, y, z)` is a call expression. So is `my_ref.f()` and `my_pointer->g(x)`.

I'm purposely handwaving away a lot of C++ esoterica here, and some overload rules are
really exceptions to this principle. But it's good to see how it works in practice.

{% highlight cpp %}
{% raw %}
template<typename T>
void f(T*) {
    puts("T*");
}
template<typename T>
void f(T) {
    puts("Just T");
}
int main() {
    int* x = nullptr;
    f(x);             // Prints "T*"
    int y = 0;
    f(y);             // Prints "Just T"
}
{% endraw %}
{% endhighlight %}

When we call `f(x)`, the compiler has two options: it could use the `void f(T)` overload,
with `T = int*`, or it could use the `void f(T*)` overload, with `T = int`. But because
`T*` is "more specific" than `T`, that's the one it tries to use first. It's able to match
up `T = int`, so it picks the first overload.

For `f(y)`, it still tries to use `void f(T*)` first, but since there's no type `T` such
that `T* = int`, it moves on to the "less specific" `void f(T)`. Now it can match up `T =
int`, so that's the overload it uses.

## The Problem

This seems nice, right? The compiler _should_ pick whichever overload most closely matches
the parameters we pass in. But what if the better choice from the compiler's perspective
is not the best choice from our perspective?

