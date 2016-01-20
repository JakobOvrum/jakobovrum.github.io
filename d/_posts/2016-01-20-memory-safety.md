---
layout: post
title: Memory Safety
---

[Memory Safe](http://en.wikipedia.org/wiki/Memory_safety) is a property of code
which cannot incur memory errors when executed. Memory safety is a highly
desirable property of programs; not only can memory errors be nondeterministic
and thus hard to debug, but memory errors are infamous for causing security
vulnerabilities.

D has opt-in statically checked memory safety, granular on the level of
functions:

```d
void main() @safe {
  import std.stdio;
  writeln("hello, world");
}
```

The `@safe` function attribute signals to the compiler that this function must
be proven to be memory safe. Applying it to the `main` function is an effective
way of requiring memory safety of the whole program
<sup>[*](# "barring the presence of any `@system` module constructors/destructors")</sup>.
The memory-unsafe counterpart of `@safe` is `@system`, which is the default.

As D is a systems programming language with many low-level features, `@safe`
functions have [a number of
restrictions](http://dlang.org/spec/function.html#function-safety), including
the inability to call `@system` functions.

#### The Default
If memory safety is so desirable, why is `@system` the default? Indeed, the
majority of functions in most programs probably adhere to `@safe` whether
they're annotated as such or not, and the consensus seems to be that `@safe`
would have been a better default. But alas, D did not always have these
attributes, and `@system` is the default for reasons of backwards compatibility.

#### Applying Attributes
The good news is that applying function attributes to functions en masse is
really simple with D's flexible attribute syntax:

```d
void foo(); // Implicitly @system, the default

@safe: // Note the colon!

void bar();
struct S { void baz(); }
```

In the above module, `bar` and `S.baz` are `@safe`, and anything added after `S`
would also be `@safe`. Attribute applications can also be bounded on both sides:

```d
@safe { // from here…
  void bar();
  struct S { void baz(); }
} // … to here

void foo(); // Implicitly @system, the default
```

Nearer attribute applictions override farther ones, which is handy when most
functions in a module are `@safe` but a minority are `@system`:
```d
@safe:
void foo();
@system void baz();
void bar();
```

`foo` and `bar` are `@safe` while `baz` is `@system`. Judicious application of
all three syntaxes alleviates the inconvenience of the default.

Additionally, anonymous/lambda functions, templated functions and functions with
inferred return type (`auto` functions) will have `@safe` inferred from the body
of the function.

#### Bridging `@safe` and `@system`
As `@safe` functions cannot call `@system` functions, annotating the whole
program with `@safe` seems impractical. To solve this, there is a mechanism for
allowing `@safe` code to call `@system` code: `@trusted`.

Applying `@trusted` to a function signals to the compiler that this function's
*interface* has been manually verified by a human to be memory safe. That is,
no possible argument combination
<sup>[*](# "including implicit arguments such as the state of global variables!")</sup>
when calling this function from `@safe` code will cause a memory error.

However, unlike `@safe`, `@trusted` functions have unrestricted access to all
language features, including calling `@system` functions. Yet, **`@trusted`
functions are freely callable from `@safe` functions**. The implication is that
`@trusted` better be applied correctly, or the whole system of checked memory
safety collapses and cannot provide any guarantees. Worse, until a memory error
is encountered, hapless programmers are falsely led to believe `@safe` code is
memory safe.

#### Guidelines for `@trusted`
Using `@trusted` correctly is important. To that end, a number of
guidelines should be followed.

 0. Don't annotate functions `@trusted` that can result in memory errors for a
    given set of inputs. This defeats the whole purpose of `@safe`.
 1. Don't annotate functions `@trusted` which haven't been thoroughly vetted for
    the possibility of memory errors.
 2. Don't annotate functions with `@trusted` using the `@trusted:` or `@trusted
    { … }` syntax even if all affected functions have been vetted for memory
    safety, as future programmers editing the code can easily miss the
    attribute.
 3. If a function contains both checkable (`@safe`) and uncheckable code,
    factor the uncheckable code into a separate function. Be careful to follow
    guideline 1 when doing this - the separate function must have a memory safe
    interface like all `@trusted` functions.
 4. When separating safe from unsafe, check the standard library for components
    that already fit the bill, e.g. [minimallyInitializedArray](https://dlang.org/phobos/std_array.html#minimallyInitializedArray).
 5. Exceptions can be thrown (with
 [enforce](http://dlang.org/phobos/std_exception.html#enforce)) to further
 narrow down valid input in order to achieve a memory safe interface for
 `@trusted` functions.
 6. In general, do *not* apply `@trusted` to templated functions. See the next
 section for how to handle those.
 7. Whenever a `@trusted` function is changed, vet it for memory safety in its
    updated form.

Note that both `@safe` and `@trusted` functions don't have to be memory safe
when called from `@system` functions. Thus, pointer or reference parameters can
be assumed to point to valid, initialized memory.

#### Templates and Attribute Inference
Templated functions can be conditionally memory safe depending on the template
arguments used to instantiate the template. That is, some instantiations may be
memory safe while others may not be. This is quite common; any templated
function where a template argument can *inject code* is suspectible to this
situation.

Consider the following code:

```d
void foo(T)(T t) {
  t.bar();
}

struct SafeStruct { void bar() @safe; }
struct UnsafeStruct { void bar() @system; }
```

We can tell that `foo(SafeStruct())` is memory safe, and that
`foo(UnsafeStruct())` is not. Fortunately, so can the compiler: the former is
accepted in `@safe` functions, while the latter is rejected. If `foo` was
explicitly annotated with `@safe`, it would not be callable with `UnsafeStruct`
in *any* code because of the call to the `@system` `UnsafeStruct.bar`, hence
templated functions are more general when using attribute inference. Templated
functions that can be inferred to be `@safe` for some instantiations can be
called *`@safe`-ready*.

Attribute inference isn't just for function templates; it's also performed for
functions nested in function templates, as well as for member functions of
templated types.

Templated functions like the above `foo` *must never be annotated `@trusted`*.
When such a function cannot be automatically proven memory safe by attribute
inference (i.e. it does not abide by the rules of `@safe`), a different approach
must be taken.

#### `@trusted` and Templated Functions
Sometimes a templated function has a memory safe interface only for some
instantiations *and* uses language constructs disallowed in `@safe`, thwarting
attribute inference:

```d
void foo(T)(T t) {
  auto p = &t; // taking the address of a local variable is disallowed in @safe functions
  p.bar();
}
```

`foo` is not `@safe`-ready: All instantiations of `foo` are `@system` regardless
of `T`. Yet we can tell that the function is memory safe as long as `p.bar()` is
memory safe. To solve this, we will allow a limited exception to the rules of
`@trusted` - *`@trusted` nested functions in templated functions do not have to
have a memory safe interface as long as all calls to the function are memory
safe*. That's a mouthful, more easily demonstrated with code:

```d
void foo(T)(T t) {
  auto p = () @trusted { return &t; } ();
  p.bar();
```

Here, we apply `@trusted` to an anonymous nested function that *doesn't* have a
memory safe interface. However, we can tell that it's not called in a way that
could cause memory errors, such as by escaping the returned pointer to a global
variable. Note that the anonymous function is called immediately (the trailing
`()`) and thus only in one place. Our function is now `@safe`-ready in a way
that doesn't compromise `@safe`.

#### Conclusion
Some of these tricks apply to the other function attributes, `pure`, `nothrow`
and `@nogc`. They can be applied to functions en masse with the same syntax, and
templated functions will have these attributes inferred in the same way. Note
that there is no equivalent of `@trusted` for the other attributes.

If you find your code doesn't work with `@safe`, be very careful about applying
`@trusted`. Presumably you wanted the benefits of `@safe` to begin with, which
`@trusted` can easily break.
