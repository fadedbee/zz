![logo](logo.png?raw=true)

Drunk Octopus - It compiles, ship it
====================================

ZZ (drunk octopus) is a modern formally provable dialect of C, inspired by rust

Its main use case is code close to hardware, where we still program C out of desperation, because nothing else actually works.
You can also use it to build cross platform libraries, with a clean portable C-standard api.

A major innovative feature is that all code is formally proven by symbolic execution in a virtual machine, at compile time.

[![Build Status](https://travis-ci.org/aep/zz.svg?branch=master)](https://travis-ci.org/aep/zz)

### quick quick start

1. [install rust](https://www.rust-lang.org/tools/install) for bootstrapping
1. install an SMT solver, either [yices2](https://github.com/SRI-CSL/yices2) or [z3](https://github.com/Z3Prover/z3)
2. cd examples/hello
3. cargo run run


### how it looks

ZZ has strong rust aesthetics, but remains a C dialect at the core.


```C++
using <stdio.h>::{printf}

export fn main() -> int {
    let r = Random{
        num: 42,
    };
    printf("your lucky number: %u\n", r.gen());
    return 0;
}

struct Random {
    u32 num;
}

fn gen(Random *self) -> u32 {
    return self->num;
}

```

### the basic ideas

#### plain C ABI

ZZ emits plain C and it will always do that. It is one of the main reasons it exists.
It will always emit C into a C compiler which will then emit the binary.

most modern languages have has their own ABI, deviating from C, so you have to use glue tools to play nice with others.
with ZZ being emitted as C, all you do is include the header.

There is no stack unwinding (C++, rust), and no coroutines (go), so all code emits to plain ansi C
with no requirements towards compiler features.

ZZ works nicely with vendor provided closed source compiler for obscure systems.
Like arduino, esp32, propriatary firmware compilers, and integrates nicely into existing industry standard microkernels like zephyr, freerots, etc.

#### safety and correctness with symbolic execution

Unlike other modern languages, ZZ has raw unchecked pointers as default. Nothing prevents you from doing crazy things,
as long as you have mathematical proof that what you're doing is defined behaviour according to the C standard.

Checking is done by executing your code in SSA form at compile time within a virtual machine in an SMT prover.
None of the checks are emitted into runtime code.

The standard library is fully stack based and heap allocation is strongly discouraged.
ZZ has convenience tools to deal with the lack of flexibility that comes with that, such as checked tail pointers.

#### namespaces, autogenerated headers and declaration ordering

No modern language has headers or semantically relevant declaration order and neither does ZZ.
but since it interacts with raw C nicely, it also emits headers, and orders declarations so a c compiler likes them.

ZZ puts module namespaces into the C symbol using underscores instead of mangling.
so my::lib::hello becomes my_lib_hello, which is C convention.

### language reference

#### top level declarations: fn, struct

fn delares a function.
struct declares a struct.
nothing fancy here.

#### storage: const, static, atomic and thread_local

const and static work exactly like in rust, but with C syntax.

```C
export const uint32_t foo = 3;
static mutable float blarg = 2.0/0.3;
thread_local mutable bool bob = true;
atomic mutable int marvin = 0;
```

const is inlined in each module and therefor points to different memory in each module.
static has a global storage location, but is private to the current module.

in effect, there is no way to have declare a shared global writable variable.
ZZ has no borrowchecker, and the restriction has nothing to do with preventing multithread races.
Instead the declarations are selected so that the resulting exported binary interface can be mapped to any other language.

if you need to export a global writeable memory location (which is still a bad idea, because threads),
you can define a function that returns a pointer to the local static.

thread_local and atomic are mapped directly to the C11 keywords.
ZZ can use nicer keywords because there are no user defined names at the top level.

#### visibility: pub, export

by default all declarations are private to a module

"export" can be used to make sure the declaration ends in the final result. that is in the binary and the export header.

"pub" marks a declaration as local to the project. it is usable in other zz modules, but not exported into the resulting binary


#### mutability: const, mut

by default, everything is const. this is the opposite of C. the mut keyword is used to make a global variable, or function argument mutable.



#### polymorphism


ZZ follows the C model of polymorphism: any struct can be cast to the same type as its first member.
In ZZ the cast is implicit because it is always safe.


```
struct Vehicle {
    int wheels;
}

struct Car {
    Vehicle base;
}

fn allowed_entry(Vehicle *self) -> bool {
    return self->wheels <= 2;
}


fn main() {
    Car c{
        base: Vehicle {
            wheels: 4,
        }
    };
    assert(!c.allowed_entry());
}


```


#### where and model

ZZ requires that all memory access is mathematically proven to be defined.
Defined as in the opposite of "undefined behaviour" in the C specification.
In other words, undefined behaviour is not allowed in ZZ.

You will quite often be told that by the compiler that something is not provable,
like indexing into an array.


this is not ok:

```C
fn bla(int * a) {
    a[2];
}
```

you must tell the compiler that accessing the array at position 2 is defined. quick fix for this one:

```C
fn bla(int * a)
    where len(a) == 3
{
    a[2];
}
```

this will compile. its not a very useful function tho, because trying to use it in any context where the array is not len 3 will not be allowed.
here's a better example:

```C
fn bla(int * a, int l)
    where len(a) >= l
{
    if l >= 3 {
        a[2];
    }
}
```


thanks to the underlying SMT solver, the ZZ symbolic executor will know that a[2] is only executed in the case where len(a) >= l >= 3, so it is defined.

The where keyword requires behaviour in the callsite, and the model keyword declares how the function itself will behave.

```C
fn bla(int a) -> int
    model return == 2 * a
{
    return a * a;
}
```

In this simple example, we can declare that a function returns 2 times its input.
But it actually does not, so this won't compile.


### theory

we can use annotations to define states for types, which neatly lets you define which calls are legal on whic
type at a given time in the program without ANY runtime code.



```C++

thery is_open(int*) -> bool;

fn open(int mut* a)
    model is_open(a)
{
    static_attest(is_open(a));
    *a = 1;
}

fn read(int require<open> mut* a)
    where is_open(a)
    model is_open(a)
{
    return *a;
}

fn close(int mut* a)
{
    *a = 0;
}
```

the above example defines a type state transition that is legal: open -> read -> close
any other combination will lead to a compile error, such as read before open.




#### struct initialization

To prepare for type elision, all expressions have to have a known type.

```C
struct A {
    int a;
    int b;
}

fn main() {
    A a = A{
        a : 2,
    };
}
```

#### conditional compilation / preprocessor

Like in rust, the prepro is not a string processor, but rather executed on the AST  **after** parsing.
This makes it behave very different than C, even if the syntax is the same as C.

The right hand side of #if is evaluated immediately and can only access preprocessor scope.

```C
struct A {
    int a;
#if def("TEST")
    uint proc;
#elif def("MAYBE")
    int proc;
#else
    void* proc;
#endif
}
```

Every branch of an #if / #else must contain a completed statement,
and can only appear where a statment would be valid,
so this is not possible:

```C
pub fn foo(
#if os("unix")
)
#endif
```

note that even code that is disabled by conditions must still be valid syntax. It can however not be type checked,

#### a note on west-const vs east-const

ZZ enforces east-const. C is not a formally correct language, so in order to make ZZ formally correct, we have to make some syntax illegal.
In this case we sacrifice west-const, which is incosistent and difficult to comprehend anyway.

west-const with left aligned star reads as if the pointer is part of the type, and mutability is a property of the pointer (which it is).

```C++
    int mut* foo;
    foo = 0; // compile error
    *foo = 0 // valid
```

unless you want to apply mutability to the local storage named foo

```C++
    void * mut foo;
    foo = 0; // valid
    *foo = 0 // compile error
```

Coincidentally this is roughly equivalent to Rust, so rust devs should feel right at home.
Even if not, you will quickly learn how pointer tags works by following the compiler errors.

#### fntype

function pointers are difficult to do nicely but also make them emit to C code well, so they don't really exist in ZZ.
instead you declare a function to be a type instead of a concrete implementation.
An fntype is emited as typedef, and therefore becomes a regular pointer type for both the C compiler and the ZZ memory tracker;

```C++
fntype rand_t() -> int;

fn secure_random() -> int {
    return 42;
}

fn main() {
    rand_t rand = secure_random;
}

```

closures do not exist in ZZ. One reason being that the C output would be difficult to use in other raw c code.
But the biggest reason is that most usage of closures is for capturing scope state.
That only really works well with garbage collected languages, otherwise its difficult to reason about (see rust).
In ZZ we instead explicitly track all state in structs and simply use plain stateless functions.


#### metaprogramming or templates: tail variants


technically zz does not have metaprogramming. template functions blow up code size and are difficult to export as plain C types.
instead zz makes code reusable by allowing allocations of structs to be larger than their member sizes.

We call this the "tail". And it can be used to make functions on fixed size arrays reusable for other sizes.

here's the String type:

```C++
export struct String+ {
    usize   len;
    char    mem[];
}
```

A + sign behind the name indicates this type has a tail.
The tail here is mem, which is specified as array with no size.

a length function could be implemented with this signature:

```C++
fn len(String+t mut * self) {
    return t;
}
```

again, the + indicates a tail. in this case, the tail size is bound to a local variable named t,
which can be used in the function body.

when allocating a new stack variable of type String, you also allocate a tail on the same stack

```C++
    String+100 s = {0};
    string::append(&s, "hello");
    string::append(&s, "world");
    printf("%.*s", s.len, s.mem);
```

again, + means tail, but here we specify an ineteger value of exact numbers of char we would like to add.
the tail is measured in number of elements of whatever is the last unsized element in the struct, not in bytes.

String can dynamically expand within the tail memory. in this case, we append some stuff to the string, without ever allocating any heap.
simply returning from the current function will clear up any memory used, without the need for destructor ordering or signal safety.
