<!---
{
  "id": "ca904186-c00b-4eed-a8be-36c975b6f944",
  "teaches": "C Libraries: API versus ABI, Seen Through objdump",
  "depends_on": [
    "05e99c97-2969-45b9-b955-5b5a62a0786e"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-01",
  "keywords": [
    "C",
    "ABI",
    "API",
    "objdump",
    "calling convention",
    "System V",
    "nm",
    "symbol table"
  ]
}
--->

# C Libraries: API versus ABI, Seen Through `objdump`

> In this exercise you will take the `libvec2.a` archive apart with `objdump` and `nm`, watch how the compiler turns a C function call into register moves, and break the ABI on purpose by changing a parameter type in the library without recompiling the program. No code is installed, no `sudo` is required: this is an analysis exercise.

## Introduction

The lecture distinguishes two interfaces of a library:

- The **API** (Application Programming Interface) is what you read in the header: names, parameter types, return types. It is written for the compiler and for humans.
- The **ABI** (Application Binary Interface) is what the machine code agrees on: which register carries the first argument, where the return value goes, how a `struct` of two `double`s is passed, how the name of the function is spelled in the object file. It is written down in the *System V AMD64 ABI* document and nobody reads it, yet every compiled program depends on it.

The linker works exclusively at the ABI level. It sees a name such as `VEC2_dot` and a hole in `main.o` that wants that name, and it fills the hole. It does not know, and cannot know, whether both sides agree on the parameter types. That is why a stale header produced a clean build but a wrong program in the previous exercise.

In this exercise you will make the ABI visible with `objdump -d` (disassemble) and `objdump -t` (symbol table), compare optimisation levels, and then break the ABI deliberately.

> ⚠ This exercise assumes x86_64 Linux and GCC. On ARM the register names differ, the lessons do not.

### 1.1) Further Readings and Other Sources

* [System V Application Binary Interface, AMD64 Supplement](https://gitlab.com/x86-psABIs/x86-64-ABI/-/raw/master/x86-64-ABI.pdf)
* [objdump(1) man page](https://man7.org/linux/man-pages/man1/objdump.1.html)
* [An Introduction to ABIs (lecture link from Moodle, chapter 18)](https://moodle.thga.de/course/view.php?id=2448&section=18)
* [Compiler Explorer](https://godbolt.org/) for comparing output of different compilers interactively

## Tasks

### Task 1: Disassemble the Library

Start in `~/vec2` with `vec2.c` and `vec2.h` from the previous exercise (prototypes restored to `double`). Compile without optimisation and disassemble:

```bash
gcc -O0 -c vec2.c -o vec2_O0.o
objdump -d vec2_O0.o | less
```

Find `<VEC2_dot>:`. Read the instructions from top to bottom. Identify:

- where the parameters arrive (look for `%xmm0` to `%xmm3`),
- where the result is left before `ret`,
- how much stack space the function allocates.

### Task 2: Compare Optimisation Levels

```bash
gcc -O2 -c vec2.c -o vec2_O2.o
objdump -d vec2_O2.o | less
```

Compare `VEC2_dot` in both files. Count the instructions. Which parts disappeared? Did the *interface* of the function (registers in, register out) change between `-O0` and `-O2`?

Save both listings for later:

```bash
objdump -d vec2_O0.o > O0.txt
objdump -d vec2_O2.o > O2.txt
diff O0.txt O2.txt | head -40
```

### Task 3: Look at the Caller

Compile `main.c` to an object file and disassemble `main`:

```bash
gcc -O0 -c main.c -I. -o main.o
objdump -d main.o | less
```

Find the `call` to `VEC2_dot`. Just before it, which registers are loaded? Just after it, which register is read? Compare with your findings from Task 1. The caller and the callee never saw each other's source code, yet they agree. That agreement is the ABI.

Also note the address printed next to `call`: it is `0` or a placeholder. Run

```bash
objdump -r main.o
```

to see the *relocation* entries, the holes that the linker will fill.

### Task 4: Symbol Tables

```bash
objdump -t vec2_O2.o
nm vec2_O2.o
nm libvec2.a
```

For each `VEC2_` symbol note the section (`.text`), the binding (`g` for global, `l` for local) and the size. Then compile a C++ file that defines the same function and compare:

```bash
printf 'struct vec2 { double x, y; };\ndouble VEC2_dot(vec2 a, vec2 b) { return a.x*b.x + a.y*b.y; }\n' > vec2pp.cpp
g++ -c vec2pp.cpp
nm vec2pp.o
nm -C vec2pp.o
```

The C++ compiler *mangles* the name. The C linker would never find it. This is why C headers meant for C++ users wrap everything in `extern "C"`.

### Task 5: Break the ABI on Purpose

Change the implementation only. In `vec2.c` turn

```c
double VEC2_dot(vec2 a, vec2 b)
```

into

```c
double VEC2_dot(vec2 a, vec2 b, int scale) {
    return scale * (a.x * b.x + a.y * b.y);
}
```

Rebuild the archive, but **do not** touch `vec2.h` and **do not** recompile `main.c`:

```bash
gcc -O2 -c vec2.c
ar -crs libvec2.a vec2.o
gcc main.o -L. -lvec2 -lm -o main
./main
```

It links. Run it several times. Which value does `scale` have? Where does it come from? Disassemble the new `VEC2_dot` and find the register that is read for `scale` but never written by the caller.

### Task 6: Break the ABI More Subtly

Undo Task 5. Now change the **struct** in `vec2.h` to `float` instead of `double`, recompile **only** the library, and link the old `main.o` against it:

```bash
sed -i 's/double x;/float x;/; s/double y;/float y;/' vec2.h
gcc -O2 -c vec2.c
ar -crs libvec2.a vec2.o
gcc main.o -L. -lvec2 -lm -o main
./main
```

Compare with the original output. Then disassemble both versions of `VEC2_dot` and find out why the same bits are now interpreted differently. Restore `double` afterwards.

### Task 7: The One Place the Compiler Can Help

Recompile `main.c` against the modified header from Task 5 (with the extra `int scale` in the **header** as well) and watch the compiler reject every call site with the old argument count. Then explain, in one sentence, why this check exists at compile time but not at link time.

## Questions

- Which registers carry the first and second `double` argument on x86_64? Which one carries the first `int`?
- Why did `-O2` change the body of `VEC2_dot` but not the way it is called?
- What is a relocation, and why does `main.o` need them while `main` (the executable) has far fewer?
- Why does `nm vec2pp.o` show `_Z8VEC2_dot4vec2S_` instead of `VEC2_dot`? Why does `nm -C` show something readable?
- In Task 5, what would happen if `scale` had been a `double` instead of an `int`? Which register would be read?
- A library in FORTRAN or Pascal can be called from C. Which of the two interfaces, API or ABI, makes that possible?

## Advice

`objdump` and `nm` are the tools that turn the lecture slide "API vs. ABI" from a definition into something you can see. Get used to running `nm` on every object file you are unsure about. It takes two seconds and answers most "undefined reference" questions before you even start guessing.

Tasks 5 and 6 are deliberately ugly. In real projects this situation arises whenever a library is updated but a program that uses it is not rebuilt. Shared libraries make this even more interesting, because the program is never relinked at all. That is the subject of the next exercise.

If something in the disassembly is unclear, paste the function into [Compiler Explorer](https://godbolt.org/) with `-O0` and `-O2` and hover over the instructions. It shows the mapping from C lines to assembly lines and is the fastest way to build intuition for what the ABI really is.
