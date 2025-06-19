## Table of contents
- [Solving issues in nanoMIPS lld](#solving-issues-in-nanomips-lld)
  - [abiflags section problem when rebasing from lld16 to lld20](#abiflags-section-problem-when-rebasing-from-lld16-to-lld20)
    - [Problem](#problem)

# Solving issues in nanoMIPS lld

Purpose of this document is to show how I solve problems in nanoMIPS lld shown on couple of real examples.

## abiflags section problem when rebasing from lld16 to lld20

While rebasing onto the llvm20 we encountered that several of the lld nanoMIPS (and much others as well)
tests are failing, one of them was the nanomips-abiflags-section.s test.

So let's see what was happening, and how to solve it?

Note: the build dir where the toolchain is built is build-nanomips-20, and we are located in the llvm-project directory.

### Problem

When running:
```
./build-nanomips-20/bin/llvm-lit -v ./lld/test/ELF/nanomips-abiflags-section.s
```

We get:
<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/error-abiflags-section-test.png?raw=true" />
</div>
<br />

Now this means nothing to us without checking the test example itself:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/abiflags-test.png?raw=true" />
</div>
<br />

So the actual code in the test is irrelevant, it will not be checked. We see here that we first use ```llvm-mc``` to translate two
assembly files to object files (nanomips-abiflags-section-sup.s only contains some dummy code, irrelevant for the test), and that
they have some flags added to them (+soft-float,-tlb,+crc etc.), later on the object files are linked and ```llvm-objdump``` is
used for checking the contents of the nanoMIPS.abiflags section. This whole cycle is done for several examples. We have the CHECK-SAME-FLAGS, CHECK-DIFFERENT-FLAGS and CHECK-EFLAGS case:
- CHECK-SAME-FLAGS: both object files have the same flags marked for their compiling (thus same abiflags sections, I'll touch on abiflags section later)
- CHECK-DIFFERENT-FLAGS: the second object file has somewhat different flags specified, and we want to check how this is processed in the linker -> this is the place we get our error from
- CHECK-EFLAGS: Removed .nanoMIPS.abiflags section from an object file, to see if we inherit abiflags from eflags

But before we continue, something about the .nanoMIPS.abiflags section:
- It is a specific section (mostly same with MIPS' .MIPS.abiflags section), which contains what are requirements or what is available in the application. Here we have information like what type of floating point calculation is used, what is the size of a general purpose register and whether some extensions are used (CRC for example).
- It is a section fixed in size (24 bytes), and its contents are calculated by mixing all the input object files' .nanoMIPS.abiflags sections
- More info here: http://codescape.mips.com/components/toolchain/nanomips/2019.03-01/docs/MIPS_nanoMIPS_ABI_supplement_01_03_DN00179.pdf in paragraph 5.2

So for some reason, our CHECK-DIFFERENT-FLAGS is failing so the idea is to look at the file we generate by linking these different cases (run the steps from ```llvm-lit``` manually and inspect the output).

When run manually here is what we get from the command that FileCheck checks (```llvm-objdump -s```):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/manual-test-output.png?raw=true" />
</div>
<br />

So from here we see that even though .nanoMIPS.abiflags section is not 24 bytes long, but 48 bytes. And it looks like these are two .nanoMIPS.abiflags sections concatenated, which means the algorithm for combining multiple abiflags sections is not working for some reason.

To be continued...

Change from shared_ptr to unique_ptr.
