---
layout: post
title: "Linkers: Symbol Resolution"
---

Object files have symbols. In very brief, symbol resolution selects the final symbols for the output object file of the link. Symbol resolution selects symbols based on strict rules. These rules are the main discussion of this post.

## Why do we need symbol resolution?

An object file cannot have multiple identically named non-local symbols. A toolchain linker combines multiple input object files into one single output object file. What would happen if multiple input object files contain identically named non-local symbols? Which of the identically named symbols should be used to resolve symbolic references to that name? Which symbols should be included in the dynamic symbol table? For all such things, symbol resolution comes to the rescue. 

For each group of symbols with the same name, symbol resolution selects one of these symbols using strict rules. The selected symbol is part of the output image symbol table and is used to resolve symbol references. 

All the selected symbols are included in the symbol table. When creating a dynamic executable or a shared library, the selected symbols satisfying certain criteria are also included in the dynamic symbol table. We will explore these criteria soon.

## Symbol resolution rules

Symbol resolution rules are simple yet somewhat complex. Let's explore them in detail. We will use examples as much as possible because I believe hands-on is the best way to learn computer science and appreciate the nitty-gritty details.

<!---
Add here that rules are of two themes: which symbol gets selected, how symbol properties are transformed!
--->

The symbol resolution rules are of two distinct themes:

- Rules which determine which input symbols to select when there are multiple identically named input symbols.
- Rules that describes transformation of properties of the selected symbol.

Symbol resolution rules somewhat vary for relocatable object files (ordinary object files), archive files and shared libraries. We will only explore rules for relocatable object files in this post. 

### Symbol resolution rules for relocatable object files

Symbol resolution primarily depends on symbol binding. Symbol binding can have 
values: *global*, *weak* and *local*. Apart from symbol binding, there are many subtleties
that makes symbol resolution a little complex. Let's explore them all.

#### 1) *local*

*local* symbols are the symbols with *local* symbol binding. A local symbol example:

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
static int foo = 1;

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

Let's start with some terminologies:

- *global* symbols are symbols with *global* symbol binding. A global symbol example:

  ```cpp
  int foo = 3;
  ```

- *weak* symbols are symbols with *weak* symbol binding. A weak symbol example:
  
  ```cpp
  __attribute__((weak))
  int foo = 5;
  ```

- *common* symbols are ~~spawn of the devil~~ uninitialized *global* symbols that are not part of any input section. *COM* (COMMON) placeholder
  is used for section index value of common symbols. *common* symbols are also called tentative symbols. They are relic from the old days
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

*weak* < *common* < *strong*

A *strong* symbol silently overrides *weak* and *common* symbols when they all have the same name. That is, a *strong* `foo` symbol
silently overrides *weak* `foo` and *common* `foo` symbols.

Example:

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

We get the following output on running the above script:

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

`foo` is `11` as we expected.

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



#### 5) Group sections

#### 6) Symbol visiblility

Symbol visibility does not affect at all which symbol is selected by the linker. However, if a symbol with non-default symbol visibility is selected, then the linker transforms the symbol's properties according to its symbol visibility.

<!---
Add an example here!
--->

#### Summary

