# Example LLVM-MSVC Module Integration

This repository demonstrates how native C/C++ functions can be injected into a Windows PE executable (`HelloWorld.exe`) using our [LLVM-MSVC](https://github.com/backengineering/llvm-msvc) SDK. It complements [this blog post](#) which walks through the technical details of function injection, symbol registration, and runtime execution.

> Both the original and modified `.exe` files are included for comparison.

Contact us: contact@back.engineering |
Website: https://codedefender.io

## What’s Included

The modified `HelloWorld.exe` includes custom functions and data linked into the binary post-compilation. These functions are written using standard C++ with support from our SDK and utilities like `lazy_importer` and `etl`.

### Injected Function: `HelloWorld`

This function is a simple proof-of-concept that dynamically loads `user32.dll` and displays a message box using `MessageBoxA`.

```cpp
#include <Windows.h>
#include <etl/string.h>
#include <lazy_importer.hpp>

EXPORT_C NTSTATUS HelloWorld(void* a1, void* a2, void* a3) {
  etl::string<16> str = "Hello World!";
  etl::string<16> user32 = "user32.dll";
  LI_FN(LoadLibraryA)(user32.c_str());
  LI_FN(MessageBoxA)(nullptr, str.c_str(), str.c_str(), NULL);
  return ERROR_SUCCESS;
}
```

### Module Entry Stub: init_entry_point

This function serves as a startup stub that runs all registered initialization functions, defined between special linker symbols.

```cpp
DECLARE_SYMBOL(init_table_start);
DECLARE_SYMBOL(init_table_end);

typedef void* (*init_fn_t)(void* a1, void* a2, void* a3);

EXPORT_C void* init_entry_point(void* a1, void* a2, void* a3) {
  init_fn_t* fns_begin = (init_fn_t*)SYMBOL_init_table_start;
  init_fn_t* fns_end = (init_fn_t*)SYMBOL_init_table_end;

  while (fns_begin != fns_end) {
    NTSTATUS status = (NTSTATUS)(*fns_begin)(a1, a2, a3);
    if (status != ERROR_SUCCESS)
      __fastfail(status);
    ++fns_begin;
  }

  [[clang::musttail]] return (*fns_begin)(a1, a2, a3);
}
```

### About the LLVM-MSVC SDK

We provide a robust SDK that allows you to develop and inject C/C++ code into existing binaries — ideal for:

- Anti-cheat systems
- Anti-tamper logic
- Runtime packers or protectors
- Fuzzing Harnesses
- Runtime Analysis

The SDK handles symbol resolution, init table registration, and supports C/C++/ASM workflows through our LLVM-MSVC toolchains. You can do much more than just execute functions at the startup of a program. If you're building advanced software protection mechanisms and want to integrate native code at a binary level, we’d love to talk.