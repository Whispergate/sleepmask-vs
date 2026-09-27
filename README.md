# Sleepmask-VS

This repository contains a collection of Sleepmask examples built on top of the Beacon 
Object File Visual Studio template ([BOF-VS](https://github.com/Cobalt-Strike/bof-vs)).
Sleepmask-VS is intended to function as a library, however, to support development efforts,
we have included the examples described below:

* `indirectsyscalls-sleepmask` - a BeaconGate example that uses indirect syscalls to call 
  proxied WinAPIs.
* `retaddrspoofing-sleepmask` - a BeaconGate example that spoofs the return address of 
  proxied WinAPIs.
* `draugr-sleepmask` - a BeaconGate example that uses return address spoofing and a spoofed 
 stack frame to create a 'legitimate' stack ([Draugr](https://github.com/NtDallas/Draugr))

Additionally, for testing custom call gates we have added:

* `TestSysCallApi()` - a function to unit test the [Core API](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/beacon-gate.htm)
set exposed by BeaconGate().
* `unit-test-bof` - a BOF to call every exported system call API exposed by the BOF C API 
 (i.e. BeaconVirtualAlloc). This can be run via a live Beacon to test that call gates work 
 in 'production'. The System Call API exposed to BOFs is a smaller subset of the 'Core' API.

**Note**: This repository assumes familiarity with BOF-VS. The BOF-VS project README contains
information about the Dynamic Function Resolution (DFR) macros and helper functions used
throughout this project.

## Quick Start Guide

### Prerequisites:

* An x64 Windows 10/11 development machine (without a security solution)
* Visual Studio Community/Pro/Enterprise 2022 (Desktop Development with C++ installed)
* The Clang compiler for Windows (Visual Studio Installer -> Modify -> Individual Components -> C++ Clang Compiler for Windows)

**Note:** Sleepmask-VS requires Clang to facilitate inline assembly blocks (`__asm{}`). Compilation will therefore fail if Clang has not been installed. This project has been tested on v17.0.3.

### Cloning the repo:

Sleepmask-VS includes BOF-VS as a submodule to simplify maintenance and development. Therefore, `git clone` will not download all of the files required to compile the project. `git submodule init` and `git submodule update` are also required to initialize the repository and fetch BOF-VS.

Alternatively, `git clone --recurse-submodules <sleepmask-vs>` will instruct Git to initialize and fetch BOF-VS as part of cloning Sleepmask-VS.

Note: If you download Sleepmask-VS as a zip, you will need to do the following to correctly configure the submodule dependency:
```
extract zip
git init
rm -r bof-vs
git submodule add https://github.com/cobalt-strike/bof-vs
```

### Debug

The `Debug` target builds Sleepmask-VS as an executable, which 
allows you to benefit from the convenience of debugging it within
Visual Studio. This will enable you to work at the source
code level without running the Sleepmask BOF through a Beacon.
In addition, BOF-VS provides a mocking framework to simplify
Sleepmask/BeaconGate development. For example, setupMockBeacon()
creates some mock Beacon memory and replicates the specified
malleable C2 settings:

```
int main(int argc, char* argv[]) {

    BEACON_INFO beaconInfo = bof::mock::setupMockBeacon(
        {
            .allocator = bof::profile::Allocator::VirtualAlloc,
            .obfuscate = bof::profile::Obfuscate::False,
            .useRWX = bof::profile::UseRWX::False,
            .module = "",
        });

[...]

```

It is also possible to mock Beacon's WINAPI calls. For example,
`createFunctionCallStructure()` can generate a `FUNCTION_CALL`
structure for the desired WinAPI. The output can then be passed
to either `runMockedSleepMask()`/`runMockedBeaconGate()` to replicate
Beacon's behavior:

```
[...]
    FUNCTION_CALL functionCall = bof::mock::createFunctionCallStructure(
        Sleep,         // Function pointer
        WinApi::SLEEP, // Human-readable WinAPI enum
        TRUE,          // Mask Beacon
        1,             // Number of arguments for function call
        GateArg(5000)  // Sleep time (5 seconds)
    );

    bof::runMockedSleepMask(sleep_mask, &beaconInfo, &functionCall);
    
    return 0;
}

```

### Release

The `Release` target compiles an object file for use
with Cobalt Strike. 

To use Sleepmask-VS:
1. Enable the Sleepmask (`stage.sleep_mask "true";`)
2. Enable required BeaconGate functions (`stage.beacon_gate { ... }`)
3. Compile Sleepmask-VS
4. Load `sleepmask.cna` in the Script Manager. This will create a new menu item called Sleepmask
5. Select the required Sleepmask from the drop down menu item
6. Save the configuration
7. Export a Beacon

### Logging

You can enable logging for the release build of your Sleepmask via setting the following define in `debug.h`:
```
// Controls logging for the release build
#define ENABLE_LOGGING 1
```
This will output debug information to `OutputDebugString()` and so will be visible via SysInternal's `DbgView` or via attaching a debugger (i.e. `Windbg`). The following shows debug output in `WinDbg` for the `draugr-sleepmask`:
```
SLEEPMASK: Masking Section - Address: 0000000000C9D000
SLEEPMASK: Masking Section - Address: 0000000000CA0000
SLEEPMASK: Calling INTERNETCONNECTA via DraugrGate
Calling INTERNETCONNECTA
Arg 0: 0x0000000000CC0004
Arg 1: 0x00000000000F1520
Arg 2: 0x0000000000000050
Arg 3: 0x0000000000000000
Arg 4: 0x0000000000000000
Arg 5: 0x0000000000000003
Arg 6: 0x0000000000000000
Arg 7: 0x00000000000FE9F0
ModLoad: 00007ffa`8c0c0000 00007ffa`8c0cb000   C:\Windows\SYSTEM32\WINNSI.DLL
DRAUGR: Finding suitable draugr trampoline gadget...
DRAUGR: Trampoline: 0x00007FFA8F8E6A23
DRAUGR: Trampoline func stack size: 192
DRAUGR: Invoking DraugrSpoofStub...
ModLoad: 00007ffa`91050000 00007ffa`91058000   C:\Windows\System32\NSI.dll
DRAUGR: Return value: 0x0000000000CC0008
SLEEPMASK: Unmasking Section - Address: 0000000000C40000
```

---

## Starburst (Mythic) Compatibility

Sleepmask-VS is fully compatible with the **Starburst** Mythic C2 agent. Starburst includes a built-in COFF loader that loads the compiled sleepmask object file at agent initialization and invokes it during every sleep cycle and BeaconGate-proxied WinAPI call.

### How to use with Starburst

1. Develop your sleepmask using the Visual Studio Debug target as usual (see above)
2. ZIP this entire repository folder (including the `bof-vs/` submodule)
3. In the Mythic payload builder, set **Sleep Mask** to `sleepmask_vs`
4. Upload the ZIP in the **Sleepmask-VS ZIP** file field that appears
5. Build the payload

The builder automatically cross-compiles the sleepmask source on Linux using Clang, embeds the resulting COFF object into the agent binary, and loads it at runtime. No manual compilation or `.cna` scripts are required.

**Note:** When `sleepmask_vs` is selected, Starburst's call stack spoofing (`spoof_profile`) is automatically disabled since the sleepmask manages its own call stack via BeaconGate.

### ZIP structure

The uploaded ZIP must contain both the sleepmask source and the BOF-VS headers. The builder searches for them automatically, so any of these layouts work:

```
# Zipped as the repo root
sleepmask-vs/
├── sleepmask-vs/
│   ├── your-sleepmask.cpp    <-- must define sleep_mask()
│   ├── sleepmask-vs.h
│   ├── library/
│   └── ...
└── bof-vs/
    └── BOF-Template/
        ├── beacon.h
        ├── beacon_gate.h
        ├── sleepmask.h
        └── base/
            └── helpers.h

# Or zipped as the contents directly
your-sleepmask.cpp
sleepmask-vs.h
library/
bof-vs/
└── BOF-Template/
    └── ...
```

### Entry point contract

The builder scans all `.cpp` files in the sleepmask-vs source directory for the `sleep_mask` function signature:

```cpp
void sleep_mask(PBEACON_INFO info, PFUNCTION_CALL functionCall);
```

The file name does not matter. Only one `.cpp` in the root of the source directory should define this function. The builder will find it and compile it as the entry point.

### How it works at runtime

1. **Agent init**: Starburst's COFF loader parses the embedded sleepmask object, allocates executable memory, resolves symbols (DFR_LOCAL imports via LoadLibrary/GetProcAddress), and processes relocations.

2. **Sleep cycle**: Before sleeping, Starburst builds a `BEACON_INFO` struct containing the agent's memory layout, heap records, XOR mask key, and `ALLOCATED_MEMORY` regions. It then calls `sleep_mask(info, functionCall)` with `bMask = TRUE` and a `WaitForSingleObject` call, which triggers the mask-sleep-unmask cycle.

3. **BeaconGate**: When enabled, WinAPI calls made by BOFs (e.g. `BeaconVirtualAlloc`, `BeaconVirtualProtect`) are routed through `sleep_mask(info, functionCall)` with `bMask = FALSE`, allowing the sleepmask to proxy them through indirect syscalls, return address spoofing, or any custom gate implementation.

### Development notes

- **DFR_LOCAL**: Use `DFR_LOCAL(MODULE, Function)` for any WinAPI calls within the sleepmask. These are resolved at load time by the COFF loader via `LoadLibrary`/`GetProcAddress`.
- **Unity build**: The entry point `.cpp` should include all library files inline (the default pattern via `#include "library/..."` directives). The builder compiles only the single entry point file.
- **Backslash paths**: Windows-style backslash include paths (`#include "library\gate.cpp"`) are automatically converted to forward slashes during cross-compilation. You do not need to modify your source.
- **Pre-compiled fallback**: If no ZIP is uploaded, the builder searches for a pre-compiled `.o` file in the `sleepmask-vs/` directory adjacent to the Starburst agent on disk.
