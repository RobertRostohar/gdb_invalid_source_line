# GDB Invalid Source Line

## Description

There seems to be an issue with the way GDB handles source line information of unused functions.
It seems that unused functions are still included when searching symbols which can result in incorrect source line mappings.

This can lead to a confusing debugging experience, as breakpoints may not align with the expected source code.

## Version affected

Seen in the latest GDB versions for Arm targets (Arm GNU Toolchain 14.3.Rel1) but probably present in earlier versions as well and might not be limited to Arm targets.

## Steps to reproduce

1. Create a simple C program with a test module which has multiple functions, some of which are unused.
   >Ensure that unused functions are large enough, so that they will overlap with valid code.
2. Compile and link the code with optimization flag `-O0` and flag `-ffunction-sections` to remove unused function sections.
   >Ensure that code is located starting at address 0.
   >Unused functions have also address 0 and are removed from the final image after linking.
   >However they are still used by GDB when processing symbols from source modules.
3. Start GDB and load the symbols.
4. Check the source line information for a function in the main module that overlaps with the address range of a removed unused function in the test module.
   The correct line information should be shown.
5. Set a breakpoint in an used function in the test module.
   >This will result in the test module being processed by GDB which will read symbols also for unused functions that have been removed from the final image.
6. Check again the source line information as described under step 4. The line information is typically incorrect and points to the removed unused function.

## Prebuilt reproducer

A simple test project for Cortex-M3 (Armv7-M) target is included to show the issue.

### Project structure

The project is described using the [CSolution Project Format](https://open-cmsis-pack.github.io/cmsis-toolbox/YML-Input-Format/).

Directory and files              | Description
-------------------------------- | ------------------------------------------------------------------
`README.md`                      | This file
`vcpkg-configuration.json`       | vcpkg configuration file for Arm Tools Environment
`test.csolution.yml`             | Solution configuration (CSolution Project Format)
`test`                           | Directory containing source files
`test/test.cproject.yml`         | Project configuration file (CSolution Project Format)
`test/main.c`                    | Main module with `main` function calling `test` function
`test/test.c`                    | Test module with `test` function and unused `test_unused` function
`test/startup.S`                 | Startup module (CMSIS Startup file for Cortex-M Device)
`test/system.c`                  | System module (stripped down CMSIS System file)
`test/gcc_arm.ld`                | Linker script (GNU Linker Script for Cortex-M)
`out`                            | Directory containing build output files
`out/test/ARMCM3/Debug/test.elf` | Prebuilt ELF file for the test program
`out/test/ARMCM3/Debug/test.map` | Memory map file for the test program

### Building the project (optional)

The project is already prebuilt. It can be built using the [CMSIS-Toolbox](https://open-cmsis-pack.github.io/cmsis-toolbox/). Alternatively, it can be built manually using the Arm GNU Toolchain.

### Show the issue

- Start GDB and load the symbols.

```bash
$ arm-none-eabi-gdb out/test/ARMCM3/Debug/test.elf
```

- GDB should start (show version) and print a message about reading symbols:

  ```text
  GNU gdb (Arm GNU Toolchain 14.3.Rel1 (Build arm-14.174)) 15.2.90.20241229-git
  ...
  Reading symbols from out/test/ARMCM3/Debug/test.elf...
  ```

- Check the source line information for line `4` in module `main.c`:

  ```gdb
  (gdb) info line main.c:4
  Line 4 of "./test/main.c" starts at address 0x1a6 <main+2> and ends at 0x1aa <main+6>.
  ```

  This is expected and aligned with the map file information from `test.elf.map`:

  ```text
   .text.main     0x000001a4        0x8 main.c.obj
  ```

- Set a breakpoint in line `39` ino module `test.c`:

  ```gdb
  (gdb) break test.c:39
  Breakpoint 1 at 0x1ae: file ./test/test.c, line 39.
  ```

  This will result in the test.c module being processed by GDB which will read symbols also for unused functions that have been removed from the final image.

- Check again the source line information for line `4` in module `main.c`:

  ```gdb
  (gdb) info line main.c:4
  Line 4 of "./test/main.c" is at address 0x19a <test_unused+410> but contains no code.
  ```

  The line information is incorrect and points to the removed unused function `test_unused`.
