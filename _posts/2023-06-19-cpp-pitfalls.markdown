# C++ Pitfalls

The First Amendment to the C++ standard states: _"The committee shall make no
rules that prevents C++ programmers from shooting themselves in the foot."
Also, the committee will keep adding more and more things to the language to
make it more error-prone. That's just how things work at the C++ committee.
This blog series will talk about some of the gotchas and pitfalls of the
language. Let's explore how absurd (and crazy) C++ is.

## Default arguments of virtual functions

C++, like most object oriented languages, supports virtual functions. Virtual
functions provides support for runtime polymorphism. Internally, virtual
functions uses vtables. But that's not exactly important for today's
discussion. Important thing to remember is, a call to a virtual function
is resolved at runtime.

C++, like many other languages, supports default arguments. Default arguments
are indeed incredibly useful. So, what's wrong with default arguments of virtual
functions? Well, let's see.

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
Hello World!
--------------

But why? Default arguments is a compile-time phenomenon. Compiler inserts them
in the function call at compile-time. On the other hand, virtual functions
is (of course) runtime phenomenon. Therefore, compiler takes the default arguments
from the compile-time function.
</details>

## Static order initialization fiasco

## Should you move return values?

## remove-erase idiom with an iterator