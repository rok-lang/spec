# 9.3. Package archive (.a) format

The build output of a library package `foo` is two files ([9.2.15](902-aot-pipeline.md#9215-output)).

| File        | Contents                                                                      |
|-------------|-------------------------------------------------------------------------------|
| `libfoo.a`  | A standard `ar` archive holding a single object `foo.o` with the concrete code |
| `foo.rhir`  | The interface used downstream for monomorphization and name resolution ([9.4](904-rhir-bytecode.md)) |

The two files sit in the same directory. `--extern foo=libfoo.a` and `--extern foo=foo.rhir` each locate the sibling ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)). Metadata such as version and producer is in the RHIR header ([9.4](904-rhir-bytecode.md)).

## 9.3.1. Standard ar archive

`libfoo.a` is a standard Unix `ar` archive. `ar`, `ranlib`, `nm`, `lld` and `clang` handle it without conversion.

```
ar t libfoo.a
foo.o
```

## 9.3.2. `foo.o`

A standard ELF / Mach-O / COFF object produced by the [AOT build pipeline](902-aot-pipeline.md). It contains only those of the package's [concrete fns](../000-meta/007-glossary.md#concrete-fn--generic-fn) reachable from the package's roots ([9.2.8](902-aot-pipeline.md#928-specialize-monomorphization)).

| Section   | Contents                                                    |
|-----------|-------------------------------------------------------------|
| `.text`   | Code of concrete `fn`s                                      |
| `.rodata` | `const`, literals, [ScanTable](../008-abi/802-scan-table.md) |
| `.data`   | Initial values of `static`s                                 |
| `.bss`    | Zero-initialized `static`s                                  |

ScanTables are ordinary constant data; generated registration code registers them with the runtime via `rok_gc_register_type` at process start ([8.2.13](../008-abi/802-scan-table.md#8213-registration-time)). There are no rok-specific sections.

## 9.3.3. Static .a principle

Every library output is a static `.a`. Dynamic libraries are not a standard rok output. Calling a dynamic library through [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) is allowed.

## 9.3.4. Version compatibility check

When a dependency's RHIR is read, its `version` and producer (the `rokc` that produced that RHIR) are compared with the current `rokc`. If they differ, it is an error stating what to rebuild ([9.4.9](904-rhir-bytecode.md#949-reading-and-rejection)).

## 9.3.5. Symbol exposure

The symbols an archive exposes are the global symbols of `foo.o`, following [Symbol mangling](905-symbol-mangling.md).

```
nm libfoo.a
0000000000000000 T _Y3f8a1c2b_foo_geom_distance_a7c91d8e44b2f190
0000000000000040 T _Y3f8a1c2b_foo_geom_Point_origin_b1234567...
```

## 9.3.6. Standard library

The standard library (`rok::bare` / `core` / `std`) is not distributed as separate archives. Each build compiles it from source (`--bootstrap`).

## 9.3.7. Linking

```
clang main.o libfoo.a librokr.a -o my_binary
```

The linker links only the needed symbols from the archive's objects. Functions exposed with `#[export_name]` can be called directly from C.

```c
extern uint32_t rok_compute(int32_t a, int32_t b);   // #[export_name("rok_compute")]
int main(void) { return (int)rok_compute(1, 2); }
```

```
clang main.c libfoo.a librokr.a -o my_binary
```

## 9.3.8. Stability

The archive format follows the [ABI stability policy](../008-abi/README.md); the RHIR format follows [9.4](904-rhir-bytecode.md).
