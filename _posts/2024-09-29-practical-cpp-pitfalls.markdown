---
layout: post
title: "Practical C++ Pitfalls"
---

The First Amendment to the C++ standard states: _"The committee shall make no rules that prevents
C++ programmers from shooting themselves in the foot. If anything, the language should encourage it."_
Consequently, the C++ committee keeps adding more and more things to the language to
make it more error-prone and confusing. That's just how things work at the C++ committee.
Remember, if you use C++ language, then you have signed up for this craziness.
This blog series will talk about some of the gotchas and pitfalls of the
language. Let's explore how absurd (and crazy) C++ can be.

## Default arguments of virtual functions

What do you think output of this program should be?


```cpp
#include <iostream>
#include <string>

class Base {
public:
    virtual void sayHello(const std::string &name = "World") {
        std::cout << "Hello " + name + "!\n";
    }
};

class Derived : public Base {
public:
    void sayHello(const std::string& name = "Earth") override {
        std::cout << "Hello " + name + "!\n";
    }
};

int main() {
    Base *b = new Derived{};
    b->sayHello();
}
```

Should the output be `Hello World!` or should it be `Hello Earth!`?

<details>

<summary> Click this to see the output and explanation</summary>

<code>
Hello World!
</code>

<br>
<br>

But why? Let's see.

C++ supports virtual functions. Virtual functions provides support for
runtime polymorphism. A call to a virtual function is resolved at runtime.

Default arguments is a compile-time phenomenon. Compiler inserts them
in the function call at compile-time. On the other hand, virtual functions
is (of course) runtime phenomenon. Therefore, the virtual function call uses
the default arguments added by the compiler.
</details>

## Expensive const reference

What do you think output of this program should be?

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    std::map<std::string, int> m;
    m["A"] = 1;
    m["B"] = 2;

    std::cout << "address of m[\"A\"]: " << &m["A"] << "\n";
    std::cout << "address of m[\"B\"]: " << &m["B"] << "\n";

    for (const std::pair<std::string, int> &p : m) {
        std::cout << "address of p.second: " << &p.second << "\n";
    }
}
```

The for-each loop in the above code iterates over `m` and stores elements
of `m` in a `const` reference variable `p`. It uses a `const` reference variable
to avoid any expensive copy operation. Does everything seems right so far?

<details>
<summary> Click this to see the output and explanation</summary>

<pre><code>address of m["A"]: 0x1a552f0
address of m["B"]: 0x1a55340
address of p.second: 0x7ffff7243c20
address of p.second: 0x7ffff7243c20
</code></pre>

<br>

Did you expect an output like this? If so, congratulations. You have a keen eye.

<br><br>

Continue reading if you expected address of <code>p.second</code> to repeat addresses of
<code>m["A"] and m["B"]</code>. The root cause is that <code>const std::map<std::string, int> &p</code>
is a reference to a temporary object. It is not a reference to elements in the map <code>m</code>.

<br><br>

<code>std::map<Key, Value></code> stores its elements as <code>std::pair<const Key, Value></code>.
So, in this case, <code>m</code> stores its elements as <code>std::pair<const std::string, int></code>.
<code>const std::pair<std::string, int> &</code> cannot refer to map elements because of type-mismatch.
The compiler creates a temporary object of type <code>std::pair<std::string, int></code> from the
map element of type <code>std::pair<const std::string, int></code> by using an implicit constructor of
<code>std::pair<std::string, int></code>. The const reference is referring to this
temporary object. Hence, the addresses of <code>p.second</code> are different from
<code>m["A"]</code> and <code>m["B"]</code>.

<br><br>

On modifying <code>const std::pair<std::string, int> &p</code> to
<code>const std::pair<const std::string, int> &p</code>, <code>p</code> has the right type to refer to map elements
and thus no temporary object is needed. In this case, the addresses of <code>p.second</code> are indeed the same
as the addresses of <code>m["A"]</code> and <code>m["B"]</code>.

<br><br>

To avoid such accidental copy operations, it is best to use <code>auto</code> feature of C++.
For example, we could use:

<br><br>

<code>
for (const auto &p : m)
</code>

<br><br>

<code>auto</code> automatically deduces the correct type and no accidental copying happens.
</details>

## erase-remove Idiom Surprise!

What do you think output of this program should be?

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main() {
    std::vector<int> v = {5, 2, 2, 5};
    const auto it = std::max_element(v.begin(), v.end());
    v.erase(std::remove(v.begin(), v.end(), *it), v.end());

    for (auto elem : v)
        std::cout << elem << ", ";
}
```

The above program is removing the maximum element in `v` by using
*erase-remove* idiom. Or is it?

<details>

<summary> Click to see the output </summary>

<code>
2, 5,
</code>

</details>

<details>

<summary> Click to see the explanation </summary>

<code>std::remove</code> name is a bit misleading as it does not remove anything.
<code>std::remove(v.begin(), v.end(), val)</code> shifts all elements equal to
<code>val</code> to the end of the container <code>v</code> and returns an iterator
pointing to the first occurence of <code>val</code> in the updated container.
For example:

<br><br>


<pre><code>std::vector&lt;int&gt; v = {5, 2, 2, 5};
std::remove(v.begin(), v.end(), 5);
// Prints: 2, 2, 5, 5; std::remove has shifted all the 5s to the end of v
for (const int &elem : v)
  std::cout << elem << ", ";
</code></pre>

So, why does the program in-question prints <code> 2, 5 </code> instead of
<code> 2, 2 </code>. The root cause is that <code>std::remove</code> takes the value which
has to be removed as a const reference and for <code>std::remove(v.begin(), v.end(), *it)</code>,
this value is <code>*it</code> -- an iterator pointing to an element in <code>v</code>.
The value of <code>*it</code> changes when the container is modified. <code>std::remove</code>
iterates over the elements in the provided range and shift the elements as required. When <code>std::remove</code>
shifts the 0th element, <code>5</code>, to the end of the container, then the container becomes <code>[2, 2, 5, 5]</code>.
But <code>*it</code> still points to the <code>0th</code> element of the container and that now has value <code>2</code>.
Thus, for the remaining iterations, <code>std::remove</code> shifts the elements having value <code>2</code> to the end of
the container.
</details>



## Static order initialization fiasco

## Should you move return values?
