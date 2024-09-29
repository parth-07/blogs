---
layout: post
title: "Linker Primer: Symbol Resolution - non-shared object files and linker scripts"
---

Linker enables separate compilation, that is, separating the code into modules instead of having everything together in one module. One of the key linker functionality required for separate compilation is connecting symbol references to symbol definitions. Symbol resolution, along with relocations, enables this functionality. This post describes symbol resolution and begins to unfold its behaviour.

This post describes symbol resolution for ELF linkers on Unix platforms. It does not aim to cover linking concepts for
linkers on other platforms or object file formats. This being said, most concepts discussed here are common for all linkers.

Symbol resolution is too big of a topic to cover all at once. This post will cover symbol resolution behavior for symbols from non-shared object files and linker scripts. Later posts will cover symbol resolution behavior that affect symbols from archive files, bitcode files, and shared objects. So, let's start without further ado.

We will start by discussing the need for symbol resolution. We will follow it up with what symbol resolution is, and then we will explore the symbol
resolution rules for non-shared object files and linker scripts. That's the plan for this post.

<!---
Add a graph here that describes where symbol resolution typically fits in the linking process.
--->

## Why do we need symbol resolution?

Object files cannot have multiple identically named non-local symbols. A toolchain linker combines multiple input object files into one single output object file. What would happen if multiple input object files contain identically named non-local symbols? Which of the identically named symbols should be used to resolve symbolic references to that name? Which symbols should be included in the symbol table? For all such things, symbol resolution comes to the rescue.

For each group of non-local symbols with the same name, symbol resolution selects one of these symbols using well-defined rules. The selected symbol is part of the output image symbol table and is used to resolve symbol references to that name.

![Which var should satisfy undefined var reference?]({{ site.baseurl }}/assets/images/UndefSymRefMultipleDefs.png){: width="300" style='border:2px solid #000000'}

It seems that most, if not all, of the complexity in symbol resolution is due to the same-named non-local symbols in multiple inputs.
So, why cannot we simply error-out if there are multiple non-local input symbols? We cannot because it is a crucial feature that enables
many use-cases. For example, `C` standard library contains memory-family functions such as `malloc`, however, a general memory allocation
algorithm does not fit all use-cases. It is critical for some use-cases to use a carefully-designed memory allocation algorithm. For such
use-cases, it is crucial to override the default memory-family functions, such as `malloc`, with the corresponding custom ones. Please note
that both the default `malloc` and the user-provided `malloc` must be non-local symbols. The linker needs to select the appropriate symbol and use it to resolve `malloc` references everywhere. Symbol resolution step is responsible for selecting the appropriate symbol when there are multiple
identically named non-local symbols.

<!---
Add link to 'symbol resolution - shared object files' here once it is ready.
--->
All the selected symbols are included in the symbol table. When creating a dynamic executable or a shared library, the selected symbols satisfying certain criteria are also included in the dynamic symbol table. We will talk more about dynamic linking and dynamic symbol table in a follow-up post.

## Symbol Resolution

Symbol resolution is the process of selecting symbols and transforming their properties as needed. The selected symbols form the output image
symbol table and are used to resolve symbol references.

Things are simple for local symbols, there are no surprises. Local symbols are always selected. Hence, in a translation unit,
a local symbol reference is always resolved to the local symbol definition with the same name.

<!---
Add a diagram!!!
--->

Things are not-so-simple for non-local symbol. If you are not careful, then you might be surprised. A selected non-local symbol overrides other non-local
symbols with the same name. A non-local symbol reference resolves to the selected non-local symbol definition with the same name. That means,
if a translation unit `A` defines and refers symbol `foo`, then it is possible that the symbol reference `foo` in the translation unit `A` gets resolved to
`foo` definition from an another translation unit.

<!---
Add a diagram!!!
--->

## Symbol resolution rules

Symbol resolution rules are simple, yet the results can be surprising at times. Let's explore these rules so they do not catch us by surprise. We will use examples as much as possible because I believe hands-on is the best way to learn computer science and appreciate the nitty-gritty details.

The symbol resolution rules are of two distinct themes:

- Rules which determine which input symbol to select when there are multiple identically named input symbols. We will call these rules as
  *Selection Rules*.
- Rules that describes transformation of properties of a selected symbol. We will call these rules as *Transformation Rules*.

The *Selection Rules* and *Transformation Rules* terms are crafted by me and are not official/famous by any means.

Remember that this post only focuses on rules for symbols from non-shared objects and linker scripts. Later posts will cover the other symbol resolution rules.

## Selection rules for symbols from non-shared object files and linker scripts

Selection rules primarily depends on symbol binding and whether the symbol is
defined, undefined, tentative or absolute. The complete list is much bigger as we will soon see.
In some (complex) cases symbol resolution results may seem surprising, but remember,
it will only happen if you forget a rule.

Before we begin, I want to discuss defined and undefined symbols and introduce some terms to help in our discussion

The term 'defined symbol' encompasses different kinds of symbols, which are treated slightly differently by the symbol resolution. We will assign terms to these different kinds of symbols and use them throughout the discussion where we need to be more precise than 'defined symbol'.

A defined symbol is a symbol that can be used to resolve symbol references. A local defined symbol can only resolve symbol references within its own translation unit, whereas a non-local defined symbol can resolve symbol references in any input. An undefined symbol is a symbol that is not a defined symbol. It cannot be used to resolve symbol references.

All defined symbols except absolute symbols point to a storage location.

There are 3 kinds of defined symbols:

- **section-relative defined symbols** are defined relative to a section. In other words, this means
  that the symbol points to a location within a section. This is by far the most common kind of symbol that you will see.
  For example:

  ```cpp
  int foo() { return 1; } // foo is a section-relative defined symbol
  int bar = 11; // bar is a section-relative defined symbol
  ```

  This term is crafted by me and is not official/famous by any means.

- **common symbols** are ~~spawn of the devil~~ relics from the old days of computing.

  *common* symbols do not have a value and do not have any section backing them.
  *common* symbols section index field holds `SHN_COMMON`. It is a placeholder section index,
  the section does not actually exist.

  *common* symbols are called tentative symbols because, unlike other defined symbols, they are tentatively defined.
  Yes, this means they might not get actually defined after all.

  *common* symbols are similar to section-relative symbols in that the symbol is supposed to point to a storage location.
  However, unlike the section-relative defined symbols, the compiler cannot assign storage for
  common symbols because their size is unknown until link time. The common symbol size is unknown
  until link time because of the core behaviour of common symbols: **If there are multiple identically
  named common symbols, they should all point to a common block of storage that is big enough to
  satisfy the largest member**. Of course, the linker is the one that allocates storage for common symbols.
  After all, only the linker knows about all the common symbols defined in a project, and hence, only
  the linker can determine how much memory to allocate for common symbols.

  If there are both *common* and non-common symbols with the same name and the symbol resolution selects one of
  the non-common symbols, then the linker does not allocate any storage for the common symbols with
  that name. This is akin to the *common* symbols with that name not being defined at all, as they
  never got any storage assigned to them. There is no trace of them in the output object file.

  *common* symbols are evil and they are disabled by default in both gcc and clang. Uninitialized global
  variables become *common* symbols when the source file is compiled with `-fcommon` option. *common* symbols
  should be avoided wherever possible.

  We will discuss more on symbol resolution for common symbols soon as we explore the symbol resolution rules.

  <!---
  Note how it differs from section-relative defined symbols. The compiler assigns unique storage to
  each section-relative defined symbol.


  From the symbol resolution perspective, the linker selects the largest common symbol from the
  multiple identically named common symbols and uses it to resolve symbol references.
  Additionally, the linker allocates storage for the selected common symbol. No storage is
  allocated for non-selected common symbols.

  Their size is not known until link time because multiple identically named *common* symbols are allowed and linker has to keep only one of them. This is different from typical symbol resolution that involves non-common symbols, where linker selects one of the symbol but all the other symbols are still present in the output image -- they are just not used for resolving symbol references and are not present in the symbol table.
  --->

- **absolute symbols** are not defined relative to any section. Their value is absolute
  and is used as it is.

We will discuss other concepts as we explore the rules :).

List of rules that we are going to see:

1. *local* symbols
1. Symbol binding: *global* vs *common* vs *weak*
1. *ABS* symbols
1. *strong* vs *strong*
1. *weak* vs *weak*
1. *common* vs *common*
1. *.gnu.linkonce.\** and *COMDAT* group

### 1) *local*

*local* symbols are the symbols with *local* symbol binding. A *local* symbol is only visible in the translation unit in which it is defined.
It cannot be used to satisfy symbol references in an another translation unit. A local symbol example in `C`:

```cpp
static int foo = 1;
```

By default all *local* symbols are preserved (selected) by the symbol resolution. If multiple input files
contains identically named local symbols, then each symbol is preserved in the output object file. Multiple identically named *local* symbols are
allowed in an object file.

Example:

```
#!/usr/bin/env bash

cat >fileA.c <<\EOF
static int foo = 1;

int bar() { return foo; }
EOF

cat >fileB.c <<\EOF
static int foo = 3;

int baz() { return foo; }
EOF

clang -o fileA.o fileA.c -c
clang -o fileB.o fileB.c -c
ld.bfd -o a.out fileA.o fileB.o
llvm-readelf -s a.out | grep 'foo'
```

We get the following output on running the above script:

```
+ cat
+ cat
+ clang -o fileA.o fileA.c -c
+ clang -o fileB.o fileB.c -c
+ ld.bfd -o a.out fileA.o fileB.o
ld.bfd: warning: cannot find entry symbol _start; defaulting to 0000000000401000
+ llvm-readelf -s a.out
+ grep foo
     2: 0000000000403000     4 OBJECT  LOCAL  DEFAULT     3 foo
     4: 0000000000403004     4 OBJECT  LOCAL  DEFAULT     3 foo
```

Note that, as expected, both `foo` symbols are present in the symbol table.

### 2) Symbol binding: *global* vs *common* vs *weak*

This rule determines which symbol to select when we have identically named *global*, *common* and *weak* symbols.

Let's begin with some terminologies and core concepts:

- Symbol binding can take values *global*, *weak* and *local*. *global* and *weak* symbols are visible in all inputs whereas
  *local* symbols are only visible within their translation unit.

- *global* symbols are symbols with *global* symbol binding. A *global* symbol is visible across translation units. It can satisify
  symbol references in other translation units. A *global* symbol example in `C`:

  ```cpp
  int foo = 3; // a global variable
  int bar() { return 1; } // a global function
  ```


- *weak* symbols are symbols with *weak* symbol binding. A *weak* symbol is like a *global* symbol but has less precedence than a *global* symbol.
  A *weak* symbol example in `C`:

  ```cpp
  __attribute__((weak))
  int foo = 5;  // a weak variable

  __attribute__((weak))
  int bar() { return 7; } // a weak function
  ```

- *common* symbols are evil. A *common* symbol always has *global* symbol binding, and thus all *common* symbols are *global* symbols.
  *common* symbols are only assigned storage if the symbol resolution selects them. In contrast, other non-absolute defined symbols
  always have storage and are part of the output image, irrespective of the symbol resolution result.

  Thankfully, common symbol functionality is disabled by default in modern toolchains. A common symbol example:

  ```cpp
  int foo; // and compile the file with '-fcommon'
  ```

- We will use the term *strong* symbol for *section-relative* *global* symbols. In simpler words, *global* symbols that are not
  *common* symbols are *strong* symbols.

The precedence among *strong*, *common* and *weak* is:

![strong vs common vs weak]({{ site.baseurl }}/assets/images/StrongVsCommonVsWeak.png){: width="350" style='border:2px solid #000000'}

A *strong* symbol can silently override *weak* and *common* symbols when they all have the same name. And a *common* symbol can silently override an identically named *weak* symbol.

Let's see an example:

```
#!/usr/bin/env bash

cat >strong.c <<\EOF
int foo = 11;
EOF

cat >common.c <<\EOF
int foo;
EOF

cat >weak.c <<\EOF
__attribute__((weak))
int foo = 13;
EOF

cat >main.c <<\EOF
#include <stdio.h>

extern int foo;

int main() {
  printf("foo: %d\n", foo);
}
EOF

clang -o strong.o strong.c -c
clang -o common.o common.c -c -fcommon
clang -o weak.o weak.c -c
clang -o main.o main.c -c
clang -o a.out main.o strong.o common.o weak.o
./a.out
```

Here we have a *strong* `foo`, a *common* `foo` and a *weak* `foo`.

Let's see the output:

```
+ cat
+ cat
+ cat
+ cat
+ clang -o strong.o strong.c -c
+ clang -o common.o common.c -c -fcommon
+ clang -o weak.o weak.c -c
+ clang -o main.o main.c -c
+ clang -o a.out main.o strong.o common.o weak.o
+ ./a.out
foo: 11
```

`foo` is `11` as we expected. This tells us that linker selected the *strong* `foo` symbol.

<!---
Add an example here!
--->

A *common* symbol silently overrides *weak* symbols.

<!---
Add an example here!
--->

### 3) ABS symbols

Linker scripts symbols are absolute symbols. Linker script symbols are symbols defined in a linker script,
or passed to linker using options such as `--defsym`. A linker script is used to specify the memory layout
of the output image. They are important in the embedded development world where a finer control over the
memory layout of programs is necessary. If you have any luck, then you will probably never hear about
linker scripts ever again.

Linker script symbols have higher precedence than ordinary *global* symbols (Of course, a *linker* favors *linker script symbols*).
A Linker scripts symbol can silently overrride an identically named non-absolute *global* symbol.

![abs vs strong vs common vs weak]({{ site.baseurl }}/assets/images/AbsVsStrongVsCommonVsWeak.png){: width="450" style='border:2px solid #000000'}

Let's see an example:

```
#!/usr/bin/env bash

cat >1.c <<\EOF
int foo() { return 1; }
int baz() { return 15; }
int bar() { return 17; }
EOF

cat >script.t <<\EOF
bar = 0x11;

SECTIONS {
  .text : {
    *(*text*)
    baz = 0x13;
  }
}
EOF

clang -o 1.o 1.c -c -ffunction-sections
ld.bfd -o 1.out 1.o -T script.t
llvm-readelf -s 1.out
```

In the example above, the two symbols `baz` and `bar` are defined two times: object file `1.o` and linker script `script.t`.

Let's see the output:

```
+ cat
+ cat
+ clang -o 1.o 1.c -c -ffunction-sections
+ ld.bfd -o 1.out 1.o -T script.t
+ llvm-readelf -s 1.out

Symbol table '.symtab' contains 5 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT   ABS 1.c
     2: 0000000000000013    11 FUNC    GLOBAL DEFAULT     1 baz
     3: 0000000000000000    11 FUNC    GLOBAL DEFAULT     1 foo
     4: 0000000000000011    11 FUNC    GLOBAL DEFAULT   ABS bar
```

In the output, we can see that both `bar` and `baz` have values as specified in the linker script `script.t`. From this, we can determine that
linker selected `bar` and `baz` from the linker script.

### 4) *strong* vs *strong*

It is an error to have multiple identically named *strong* symbols in a link. :).

Example:

```
#!/usr/bin/env bash

cat >1.c <<\EOF
int foo() { return 1; }
EOF

cat >2.c <<\EOF
int foo() { return 3; }

int main() {
  return foo();
}
EOF

clang -o 1.o 1.c -c
clang -o 2.o 2.c -c
clang -o a.out 1.o 2.o
```

We get the following output on running the above script:

```
+ cat
+ cat
+ clang -o 1.o 1.c -c
+ clang -o 2.o 2.c -c
+ clang -o a.out 1.o 2.o
/usr/bin/ld: 2.o: in function `foo':
2.c:(.text+0x0): multiple definition of `foo'; 1.o:1.c:(.text+0x0): first defined here
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

### 5) *weak* vs *weak*

If there are multiple identically named *weak* symbols, then the linker selects the first one as per the order in which the linker reads symbols.

It does not matter if a *weak* symbol has aninitializer or not. The below two *weak* symbols have equal precendence:

```cpp
__attribute__((weak))
int fred;
```

```cpp
__attribute__((weak))
int fred = 11;
```


But in which order does the linker reads symbols? The linker sequentially reads symbol table from input files as they appear in the link command-line. Therefore, if *fileA* appears before *fileB* in the link command-line, the linker reads symbols from *fileA* symbol table before reading symbols from *fileB* symbol table.

Example:

```
#!/usr/bin/env bash

cat >fileA.c <<\EOF
__attribute__((weak))
int foo = 1;
EOF

cat >fileB.c <<\EOF
__attribute__((weak))
int foo = 3;
EOF

cat >fileC.c <<\EOF
__attribute__((weak))
int foo;
EOF

cat >main.c <<\EOF
#include <stdio.h>

extern int foo;

int main() {
  printf("foo: %d\n", foo);
}
EOF

clang -o fileA.o fileA.c -c
clang -o fileB.o fileB.c -c
clang -o fileC.o fileC.c -c
clang -o main.o main.c -c
clang -o a.fileAFirst.out main.o fileA.o fileB.o fileC.o
./a.fileAFirst.out
clang -o a.fileBFirst.out main.o fileB.o fileA.o fileC.o
./a.fileBFirst.out
clang -o a.fileCFirst.out main.o fileC.o fileA.o fileB.o
./a.fileCFirst.out
```

We get the following output on running the above script:

```
+ cat
+ cat
+ cat
+ cat
+ clang -o fileA.o fileA.c -c
+ clang -o fileB.o fileB.c -c
+ clang -o fileC.o fileC.c -c
+ clang -o main.o main.c -c
+ clang -o a.fileAFirst.out main.o fileA.o fileB.o fileC.o
+ ./a.fileAFirst.out
foo: 1
+ clang -o a.fileBFirst.out main.o fileB.o fileA.o fileC.o
+ ./a.fileBFirst.out
foo: 3
+ clang -o a.fileCFirst.out main.o fileC.o fileA.o fileB.o
+ ./a.fileCFirst.out
foo: 0
```

### 4) *common* vs *common*

If there are multiple identically named *common* symbols, then the linker selects the common symbol with the maximum size.
Linker only allocates storage for the *selected* *common* symbols. So, if there 3 identically named *common* symbols with sizes:
`2`, `4`, and `8`. Then linker selects the common symbol that has size `8` and allocates storage for it. The non-selected *common* symbols
are not assigned a storage and are thus not contained in the output image.

Example:

```
#!/usr/bin/env bash

cat >fileA.c <<\EOF
int foo;
EOF

cat >fileB.c <<\EOF
int foo[2];
EOF

cat >fileC.c <<\EOF
int foo[4];
EOF

clang -o fileA.o fileA.c -c -fcommon
clang -o fileB.o fileB.c -c -fcommon
clang -o fileC.o fileC.c -c -fcommon
ld.bfd -o a.out fileA.o fileB.o fileC.o -Map a.map.txt
llvm-readelf -s a.out | grep 'foo'
head a.map.txt -n 5
```

We get the following output on running the above script:

```
+ cat
+ cat
+ cat
+ clang -o fileA.o fileA.c -c -fcommon
+ clang -o fileB.o fileB.c -c -fcommon
+ clang -o fileC.o fileC.c -c -fcommon
+ ld.bfd -o a.out fileA.o fileB.o fileC.o -Map a.map.txt
ld.bfd: warning: cannot find entry symbol _start; defaulting to 0000000000401000
+ llvm-readelf -s a.out
+ grep foo
     4: 0000000000401000    16 OBJECT  GLOBAL DEFAULT     1 foo
+ head a.map.txt -n 5

Allocating common symbols
Common symbol       size              file

foo                 0x10              fileC.o
```


### 6) *.gnu.linkonce.\** and *COMDAT* group

*.gnu.linkonce.\** and *COMDAT* group are features designed to remove code duplication. Many C++ features such as templates
and inline functions results in the same fragments of code in multiple translation units. For example, if a template
`template <typename T> T foo(T t)` is instantiated in multiple translation units for `T = int`, then all of these translation
units have duplicate code for the function `template <> int foo(int t)`. There are two problems with this:

- Multiple identically named *global* *section-relative* symbols cause `multiple definition error`.
- The final object file generated by the linker will have code duplication
  because the linker merges input section contents to form output sections.

The first issue can be resolved by making the symbols *weak*. Multiple identically named *weak* symbols do not cause
`multiple definition error`. Linker picks the first definition it sees. All seems good. However, this does nothing for
the code duplication in the output object file. Non-selected symbols are still present in the output image -- they are just
not part of the symbol table and are not used for resolving symbol references. `*.gnu.linkonce.\** and *COMDAT* group features
fixes both the issue. Let's see what they are all about.


#### *.gnu.linkonce.\**

**As the name suggests, it is a GNU extension. LLD does not support *.gnu.linkonce.\** functionality.**

*.gnu.linkonce.\** feature instructs the linker that only one of the sections having the name *.gnu.linkonce.<SomeName>*
should be included in the output image and the remaining sections should be discarded. When the link contains multiple sections have
the name *.gnu.linkonce.<SomeName>*, then the linker keeps the first one it sees and discards the rest. For example:

```
#!/usr/bin/env bash

cat >1.c <<\EOF
__attribute__((section(".gnu.linkonce.foo")))
int foo() { return 11; }
EOF

cat >2.c <<\EOF
__attribute__((section(".gnu.linkonce.foo")))
int foo() { return 1; }
EOF

cat >main.c <<\EOF
int foo();
int baz();
int main() {
  return foo();
}
EOF

clang -o 1.o 1.c -c
clang -o 2.o 2.c -c
clang -o main.o main.c -c
clang -o main.2.out main.o 1.o 2.o
./main.1.out
clang -o main.2.out main.o 2.o 1.o
./main.2.out
```

Executing `./main.1.out` prints `11` and executing `main.2.out` prints `1`. This makes sense because
the linker keeps the first section it sees with the name `.gnu.linkonce.foo`. The linker first sees
`1.o(.gnu.linkonce.foo)` when generating `main.1.out`. On the other hand, the linker first sees
`2.o(.gnu.linkonce.foo)` when generating `main.2.out`.

#### *COMDAT* group

*COMDAT* group is an improvement over the `.gnu.linkonce.*` feature. *COMDAT* group is one of the functionality
of the group section feature. A *COMDAT* group section has one or more members, where members are simply sections of the
object file containing the *COMDAT* group section. If multiple *COMDAT* groups have the same signature symbol, then
linker keeps members from one of the group and discards the members from the remaining groups. Group section feature
and *COMDAT* group will be discussed in detail in a later post.

A *COMDAT* group example:

```
#!/usr/bin/env bash

cat >foo.h <<\EOF
template<typename T>
T foo(T u) { return u; }
EOF

cat >1.cpp <<\EOF
#include "foo.h"

int bar() { return foo(1); }
EOF

cat >2.cpp <<\EOF
#include "foo.h"

int baz() { return foo(3); }
EOF

cat >main.cpp <<\EOF
int bar();
int baz();

int foo(int u) { return 17; }

int main() { return bar() + baz(); }
EOF

clang++ -o 1.o 1.cpp -c
clang++ -o 2.o 2.cpp -c
clang++ -o main.o main.cpp -c
clang++ -o main.out main.o 1.o 2.o -stdlib=libc++
```

`llvm-readelf -S 1.o` prints:

```
There are 12 section headers, starting at offset 0x2a0:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .strtab           STRTAB          0000000000000000 000221 00007c 00      0   0  1
  [ 2] .text             PROGBITS        0000000000000000 000040 000010 00  AX  0   0 16
  [ 3] .rela.text        RELA            0000000000000000 0001d8 000018 18   I 11   2  8
  [ 4] .group            GROUP           0000000000000000 000140 000008 04     11   5  4
  [ 5] .text._Z3fooIiET_S0_ PROGBITS     0000000000000000 000050 00000c 00 AXG  0   0 16
```

`G` flag in `.text._Z3fooIiET_S0_` row indicates that this section is a group member. Similarly, `.text._Z3fooIiET_S0_`
section in `2.o` is also a group member. The groups in `1.o` and `2.o` have the same signature, that is, the section name `.text._Z3fooIiET_S0_`.
The linker discards members from one of the groups because the groups have the same signature, and thus removes the code duplication.

### Summary

<TODO>

