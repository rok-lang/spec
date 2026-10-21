# 1.10. `rok::bare::mlir::plugin` — registering user-defined dialects

`rok::bare::mlir::plugin` defines the interface through which the rok compiler dynamically loads externally built MLIR dialects and adds them as call targets of [`__mlir_op`](109-mlir.md). It is for library authors who want to layer domain-specific compilers (cryptography, signal processing, formal verification, new accelerator ISAs, etc.) on top of rok.

This section defines the **registration mechanism and safety responsibilities** of plugins. The definition of a dialect itself (the meaning and schema of its ops / types / attributes) belongs to MLIR (TableGen / IRDL) and is outside the scope of this specification.

## 1.10.1. Separation of areas — what is inside / outside the spec

| Responsibility                                           | Where                                      |
|----------------------------------------------------------|--------------------------------------------|
| Dialect definition (op schema, type system)              | **Outside the spec** — MLIR (TableGen `.td` / IRDL) |
| Dialect build (`.so` / `.dylib` / `.dll`)                | **Outside the spec** — user build tools    |
| Interface by which the rok compiler loads / registers a plugin | **Inside the spec** (this section)   |
| Calling loaded plugin ops with `__mlir_op`               | **Inside the spec** ([1.9](109-mlir.md))   |
| Lowering / passes of plugin ops                          | **Outside the spec** — the plugin's own responsibility |

This separation keeps rok independent of the evolution of MLIR's own dialect-definition DSLs (TableGen → IRDL → next-generation formats).

## 1.10.2. Plugin build form

`rokc` embeds MLIR ([0.2.4](../../refs-lang/000-meta/002-pipeline.md#024-backends)) and loads plugins into its own MLIR context. A plugin is a dynamic library following MLIR's standard plugin form.

| Kind            | Entry point (C ABI)               |
|-----------------|-----------------------------------|
| Dialect         | `mlirGetDialectPluginInfo()`      |
| Pass (optional) | `mlirGetPassPluginInfo()`         |

The dialect plugin's namespace goes in the `NS` position of `__mlir_op::<"NS.op_name", ...>`. A plugin must be built against the LLVM version embedded in `rokc`. The plugin uses `rokc`'s MLIR instead of carrying its own: its MLIR / LLVM symbols are left unresolved and bind at load time to the ones `rokc` exports. `rokc` exports the MLIR it links (the dialects of [refs-lang 8.7.1](../../refs-lang/008-abi/807-mlir-dialects.md#871-dialects-used-by-ordinary-code-recap) and [8.7.5](../../refs-lang/008-abi/807-mlir-dialects.md#875-library--acceleration-domain-dialects) and the infrastructure they use) and LLVM's support library; a plugin may build on those and must carry anything else (for example other upstream dialects) itself. A plugin linked against its own copy of MLIR registers into registries `rokc` does not see. On Windows `rokc.exe` exports these symbols through an import library shipped with the toolchain's MLIR plugin development files; a plugin DLL links against it, uses the same C runtime as `rokc` (the dynamic release CRT) and exports its entry points. A DLL cannot reach data `rokc.exe` exports except through `__declspec(dllimport)`; the plugin development headers arrange this for MLIR's explicit TypeIDs, and other such data is not available to plugins. Exact build flags are implementation-defined.

## 1.10.3. Plugin loading

There are two ways to specify a plugin. Either way, `rokc` loads it before lowering.

### CLI options

```bash
rokc build --load-mlir-plugin /path/to/libmydialect.so main.rok
rokc build --load-mlir-plugin mydialect main.rok         # look up by name
rokc build --load-mlir-pass-plugin mypasses --mlir-pass my-lower main.rok
```

| Option                                 | Effect                                          |
|----------------------------------------|-------------------------------------------------|
| `--load-mlir-plugin <path\|name>`      | loads a dialect plugin                          |
| `--load-mlir-pass-plugin <path\|name>` | loads a pass plugin                             |
| `--mlir-pass <pass>`                   | runs `<pass>` before conversion to the LLVM dialect |

All may be repeated. A `name` that is not a path is looked up as the host's library file name (`lib<name>.dylib` or `lib<name>.so` on macOS, `lib<name>.so` on Linux, `<name>.dll` on Windows) in the directories of `ROK_MLIR_PLUGIN_PATH` (lookup details are implementation-defined). If a plugin fails to load, it is a build error. Plugin files are part of the incremental build fingerprint ([refs-lang 9.2.16](../../refs-lang/009-tools/902-aot-pipeline.md#9216-incremental-build)), and `rok` passes these options to `rokc` unchanged.

### Package manifest declaration

A plugin declared in the package manifest (`rok.rokon`, [refs-lang 9.10](../../refs-lang/009-tools/910-rok-frontend.md)) is loaded automatically at build time. The manifest format is defined by [refs-lang 9.10.3](../../refs-lang/009-tools/910-rok-frontend.md#9103-rokrokon--roklock). The field for declaring plugins is not yet part of that format; once decided, it will be added to 9.10.3.

## 1.10.4. Calling plugin ops

Ops of a loaded plugin are called with the same syntax as upstream MLIR ops.

```rok
use rok::bare::mlir::__mlir_op;

unsafe {
    // assume the plugin namespace is "crypto"
    let cipher: Vector<u8, 16> = __mlir_op::<"crypto.aes_round", _>(
        plaintext, key,
        attributes = { "round_index": __mlir_attr::<"3 : i32"> },
    );
}
```

All call syntax of [1.9 rok::bare::mlir](109-mlir.md) (attributes / regions / variadic) applies as is. The op name and signature are verified by the MLIR context that loaded the plugin.

## 1.10.5. Safety responsibilities

The responsibilities borne by plugin users (library authors + users of those libraries) are:

- **LLVM version match**: the plugin must be built against the LLVM version embedded in `rokc` and bind to `rokc`'s exported MLIR / LLVM symbols (1.10.2); otherwise loading it fails.
- **Op semantic verification**: MLIR checks op names and signatures. The semantics / side effects / GC safety of ops are the plugin author's responsibility.
- **GC root tracking**: when a plugin op handles GC heap objects, the plugin's lowering must uphold root tracking that keeps those pointers in shadow stack frame slots ([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)); otherwise the objects may be reclaimed in a GC cycle.
- **Target availability**: if a plugin supports only specific targets (GPU / FPGA / quantum hw), calling it in a build without that target enabled is an error at the lowering stage.

## 1.10.6. ABI version policy

The plugin ABI is not defined by rok; it follows MLIR's plugin API (`MLIR_PLUGIN_API_VERSION`). Compatibility is judged by MLIR, and plugins must be rebuilt when the embedded LLVM version changes. This matches the general principle that the `rok::bare` tier is not stable ([0.5 Standard library tiers](../../refs-lang/000-meta/005-standard-libraries-tiers.md)).

## 1.10.7. Build isolation

Dialects registered as plugins are **active only in that build**. The surface semantics of rok code written by ordinary users do not depend on whether a plugin is present.

```rok
// built without the crypto plugin, this is a build error (MLIR does not know the op)
unsafe {
    __mlir_op::<"crypto.aes_round", _>(...)
}
```

`rokc` cannot know a plugin's namespace, so op names are checked by MLIR, and its diagnostic is reported as a build error.

This isolation keeps plugin dialects from intruding on the semantics of the rok language itself. The meaning of ordinary user code is always determined by this specification alone.

## 1.10.8. Comparison with other languages

| Language   | Plugin dialect mechanism                            | Where      |
|------------|-----------------------------------------------------|------------|
| Mojo       | Integrated at compiler build time (external plugin loading not formalized) | Build |
| MLIR C API | `mlirContextLoadAllAvailableDialects` etc.          | Library    |
| LLVM       | `-load=plugin.so` (legacy pass manager)             | CLI        |
| **rok**    | **`--load-mlir-plugin ...` → loaded into `rokc`'s MLIR** | CLI   |

rok defines no plugin ABI of its own and uses MLIR's plugin form as is.

## 1.10.9. Out of scope

The following are outside the scope of this specification.

- Defining dialects / ops inside rok source code (Mojo does not do this either)
- Loading two plugins with the same namespace together (conflict — rejected by MLIR)
- Adding dialects at runtime (execution time)
- Compatibility of plugins across LLVM versions (follows the MLIR plugin API)
