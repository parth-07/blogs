---
layout: post
title: "Linker Primer: Symbol Resolution - non-shared object files and linker scripts"
---

Linker enables separate compilation, that is, separating the code into modules instead of having everything together in one module. One of the key linker functionality required for separate compilation is connecting undefined symbol references to symbol definitions. Symbol resolution, along with relocations, enables this functionality. This post describes symbol resolution and begins to unfold its behaviour.

Symbol resolution is too big of a topic to cover all at once. This post will cover symbol resolution behavior for symbols from non-shared object files and linker scripts. Later posts will cover symbol resolution behavior that affect symbols from archive files, bitcode files, and shared objects. So, let's start without further ado.

Symbol resolution does two things:

- Selects symbols. Selected symbols form the output image symbol table and satisfy undefined symbol references.
- Transforms the selected symbols. 

<!---
Add a graph here that describes where symbol resolution typically fits in the linking process.
--->

## Why do we need symbol resolution?

Object files cannot have multiple identically named non-local symbols. A toolchain linker combines multiple input object files into one single output object file. What would happen if multiple input object files contain identically named non-local symbols? Which of the identically named symbols should be used to resolve symbolic references to that name? Which symbols should be included in the symbol table? For all such things, symbol resolution comes to the rescue. 

For each group of symbols with the same name, symbol resolution selects one of these symbols using well-defined rules. The selected symbol is part of the output image symbol table and is used to resolve symbol references to that name. 

![Which var should satisfy undefined var reference?](/assets/images/UndefSymRefMultipleDefs.png){: width="300" style='border:2px solid #000000'}

<!---
Add link to 'symbol resolution - shared object files' here once it is ready.
--->
All the selected symbols are included in the symbol table. When creating a dynamic executable or a shared library, the selected symbols satisfying certain criteria are also included in the dynamic symbol table. We will talk more about dynamic symbol table in a follow-up post.

## Symbol resolution rules

Symbol resolution rules are simple, yet the results can sometimes be surprising. Let's explore these rules so they do not surprise us. We will use examples as much as possible because I believe hands-on is the best way to learn computer science and appreciate the nitty-gritty details.

The symbol resolution rules are of two distinct themes:

- Rules which determine which input symbol to select when there are multiple identically named input symbols. We will call these rules as
  *Selection Rules*.
- Rules that describes transformation of properties of a selected symbol. We will call these rules as *Transformation Rules*.

Remember that this post only focuses on rules for symbols from non-shared objects and linker scripts. Later posts will cover the other symbol resolution rules.

### Selection rules for symbols from non-shared object files and linker scripts

Selection rules primarily depends on symbol binding and whether the symbol is defined, undefined, tentative or absolute. Symbol binding can have values: *global*, *weak*, and *local*. Some subtleties can make symbol resolution results seem surprising, but that's only if you forget a rule.

List of rules that we are going to see:

1. *local*
1. *global* vs *common* vs *weak*
1. *strong* vs *strong*
1. *weak* vs *weak*
1. *common* vs *common*
1. *ABS*
1. *group* sections

#### 1) *local*

*local* symbols are the symbols with *local* symbol binding. A *local* symbol is only visible in the translation unit in which it is defined.
It cannot be used to satisfy undefined symbol references in an another translation unit. A local symbol example in `C`:

```cpp
static int foo = 1;
```

By default all *local* symbols are preserved (selected) by the symbol resolution. If multiple input files
contains identically named local symbol, then each symbol is preserved in the output object file. Multiple identically named *local* symbols are
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

Note that, as expected, both `foo` symbols are retained in the symbol table.

#### 2) *global* vs *common* vs *weak*

This rule determines which symbol to select when we have *global*, *common* and *weak* symbols of the same name in a link.

Let's begin with some terminologies:

- *global* symbols are symbols with *global* symbol binding. A *global* symbol is visible across translation units. It can satisify an undefined 
  symbol reference in other translation units. A *global* symbol example in `C`:

  ```cpp
  int foo = 3;
  ```


- *weak* symbols are symbols with *weak* symbol binding. A *weak* symbol is like a *global* symbol but has less precedence than a *global* symbol.
  A *weak* symbol example in `C`:
  
  ```cpp
  __attribute__((weak))
  int foo = 5;
  ```

- *common* symbols are ~~spawn of the devil~~ uninitialized *global* symbols that are not allocated . *COM* (COMMON) placeholder
  is used for section index value of common symbols. They are placed in an output section by the linker. *common* symbols are also called tentative symbols. They are relic from the old days
  of compiling and linking. Thankfully, common symbol functionality is disabled by default in modern toolchains. A common symbol example:

  ```cpp
  int foo; // and compile the file with '-fcommon'
  ```

<!---
Add example of common symbol here.
--->

- We will use the term *strong* symbol for *global* symbols that are defined in an input section. In simpler words, *global* symbols that are not
  *common* symbols are *strong* symbols.

The precedence among *strong*, *common* and *weak* is:

![strong vs common vs weak](/assets/images/StrongVsCommonVsWeak.png){: width="350" style='border:2px solid #000000'}

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

#### 3) *strong* vs *strong*

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

#### 3) *weak* vs *weak*

If there are multiple identically named *weak* symbols, then the linker selects the first one as per the order in which the linker reads symbols. It does not matter if a *weak* symbol has initializer or not. The below two *weak* symbols have equal precendence:

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

#### 4) *common* vs *common*

If there are multiple identically named *common* symbols, then the linker selects the common symbol with the maximum size.

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

#### 5) Linker script symbols

Linker script symbols are symbols defined in a linker script, or passed to linker using options such as `--defsym`. A linker script is used to specify the memory layout of the output image. They are important in the embedded development world where a finer control over the memory layout of programs is necessary. If you have any luck, then you will probably never hear about linker scripts ever again. 

Linker script symbols have higher precedence than ordinary *global* symbols (Of course, a *linker* favors *linker script symbols*). A Linker scripts symbol can silently overrride an identically named object file *global* symbol.

![abs vs strong vs common vs weak](/assets/images/AbsVsStrongVsCommonVsWeak.png){: width="450" style='border:2px solid #000000'}

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


#### 6) Group sections

### Transformation rules for symbols from non-shared object files and linker scripts

#### Symbol visiblility

Symbol visibility does not affect at all which symbol is selected by the linker. However, if a symbol with non-default symbol visibility is selected, then the linker transforms the symbol's properties according to its symbol visibility.

<!---
Add an example here!
--->

#### Summary

