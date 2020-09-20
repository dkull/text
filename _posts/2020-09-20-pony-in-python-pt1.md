---
layout: post
title:  "Using Pony from Python"
date:   2020-09-20 19:28:00 +0300
categories: pony python compilation ffi
---

Using Pony code from Python would be really useful. So here's how to do it.
This guide is based on Pony `0.37.0` built from source with Clang. Build config uses no special flags.

# Pony library

This is an example Pony logic we would like to call from Python. Let's name it `foo.pony`
```pony
actor@ Foo
  var counter: USize = 0

  new create() =>
    None

  fun val get_counter(): USize =>
    counter

  be greet_and_increment() =>
    @printf[I32]("Hi nr. %d\n".cstring(), counter)
    counter = counter + 1
```

This needs to be compiled into a static library:
```bash
pony -l
```
This created `foo.h` and `libfoo.a`
Since there seems to be an issue with Pony `0.37.0` we need a little helper C file. Let's name it `helper.c`

```c
// https://github.com/ponylang/ponyc/issues/3464
void Main_runtime_override_defaults_oo(void* opt)
{
  (void)opt; // avoid warning/error about unused variable
  return;
}
```

Let's setup our build environement:

```bash
export PONYRT_INCLUDE=<PONYBUILDIR>/src/libponyrt/
export PONYRT_COMMON=<PONYBUILDDIR>/src/common/
export PONYRT_LIB=<PONYBUILDDIR>/build/release/libponyrt-pic.a
```

And now the build command:

```
# -shared -> we want a shared library
# -o libfoo.so -> the output filename
# -I ... -> the included libraries
# -Wl,--whole-archive -> start export of all symbols
# libponyrt-pic.a -> Pony runtime
# libfoo.a -> our static library
# helper.c -> our helper code
# -Wl,--no-whole-archive -> end export-all, required
# extra linked stuff, not required for this example
gcc -shared -o libfoo.so -I. -I $PONYRT_INCLUDE -I $PONYRT_COMMON -Wl,--whole-archive ../ponyc/build/release/libponyrt-pic.a libfoo.a helper.c -Wl,--no-whole-archive -lpthread -ldl
```

We now have `libfoo.so`. `readelf -s libfoo.so` should confirm we have exported `Foo*` and `pony_*`

And now to use the shared library we have to import it with LoadLibrary and call some Pony runtime setup functions.
Setting argtypes and restypes is also required.

```python
from ctypes import *

lib = cdll.LoadLibrary('./libfoo.so')

lib.pony_init.argtypes = [c_int, c_char_p]
lib.Foo_Alloc.restype = c_void_p
lib.Foo_tag_hi_o__send.argtypes = [c_void_p]
lib.Foo_val_get_counter_Z.argtypes = [c_void_p]

lib.pony_init(1, c_char_p(b'name'))
lib.pony_start(True, None, None)
ptr = lib.Foo_Alloc()

for x in range(100):
    lib.Foo_tag_hi_o__send(ptr)

# pony actors are async, give a moment for them to finish
time.sleep(0.1)

assert lib.Foo_val_get_counter_Z(ptr) == 99
```