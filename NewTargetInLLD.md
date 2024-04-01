# Adding a new ELF Machine Target to LLD

In this blog I'll cover a method to add new ELF Machine Targets to lld, then I'll talk about implementing target specific relaxations (or transformations). I'll also talk about implementation of nanoMIPS target.

## Why LLD

So the first question that comes to me, is why should we use lld when we have gnu linkers (bfd and gold) that are working fine. What people like to emphasize the most is lld's speed. On average lld runs twice as fast as the GNU gold linker, but it tends to use more memory. It is small, it's source code is several times smaller in term of code line count. But what I find to be most helpful for developers are more flexible data structures and algorithms then on gnu's gold linker. The code is also easier to understand then in gold, so it is easier for developers to work on it further. Lastly, it works good with clang as clang can generate bitcode object files which can be then sent to lld to perform link time optimizations on whole programs.

I hope that this explains the whys behind lld's popularity.

## Useful structures in LLD

In this section, I'll briefly describe structures in lld that were useful to me, when it comes to implementing a new Machine Target to LLD.
- ```InputSection``` - represents a section from an object file or a synthetic one that should be somehow put in the output file or discarded. One can easily traverse its relocations and access its data.
- ```OutputSection``` - represents a section in an output file containing ```InputSection```s from various sources.
- ```Symbol``` - represents a symbol from an object file, archive or linker defined symbols.
- ```Relocation``` - represents a relocation in a ```InputSection```, with its type, expression, offset in ```InputSection``` and ```Symbol``` to which the ```Relocation``` refers
- ```Writer``` - A class that mainly writes the data from ```OutputSection```s to the output file
- ```Driver``` - Main driver of the lld linker, it is responsible for controlling the linker's flow of work
- ```Options.td``` - Not a structure itself, but it defines options that can be passed to the linker, most of the time those option values are stored in ```Config``` structure

## LLD linking steps

Here, we'll quickly walk through steps of linking with **lld**. 
1. First **lld** tries to deduce which linker should be used (**elf** linking, **coff** linking, **macho** linking or **wasm** linking), it is usually deduced by calling a specific linker instead of just **lld** (**ld.lld** is the linker for **Unix**) or by passing the -flavor option (**gnu** if the same outcome as **ld.lld** is expected).
2. We are talking about **elf** linking from now on, so after we've determined that we want to use the **elf** linker initialization of needed structures occurs (configuration, script, context, etc.) and the ```linkerMain``` is called.
3. In ```linkerMain``` arguments passed to the linker are parsed, and most commonly somehow remembered in the ```Config``` structure. Also, linker tries to find all the needed files, libraries and creates their structures before ```link``` is called.
4. After processing some specific arguments (if passed), files and libraries passed to the linker are parsed, creating needed ```InputSection```s and filling the symbol table in ```parseFile``` and ```initSectionsAndLocalSyms```. When parsing is completed for all the files, then we check for duplicate symbols in the ```postParseObjectFile``` function. After no new symbols are there to be added (except of some synthetic ones), some target dependent properties are set, such as ```eflags```, ```maxPageSize```, etc. Then, sections are copied into partitions (those that we want in the output file) and synthetic sections are created. Next, the linker script is read for section commands and through ```finalizeInputSections``` function of ```OutputSection``` ```InputSection```s are added to the corresponding ```OutputSection```s. Finally ```writeResult``` is called.
5. ```writeResult``` runs the ```Writer```. Firstly, ```finalizeSection``` is invoked. This function scans relocations, adds symbols to the final symbol table, creates a list of ```OutputSection```s, finalizes synthetic sections, assigns final value to some address dependent symbols, changes data in some sections (relaxations, thunks) and fills out section headers. At the end ```writeHeaders``` and ```writeSections``` take place and write the program to the output file, resolving relocations also happens here.

## Adding a new ELF Machine Target

Simply said, all you need to do to add a new machine target to lld is implement a class representing your architecture as a derivation of lld's TargetInfo class. Easier said than done. To achieve this you need to implement two functions:
-  ```getRelExpr``` - Used to return the sort of relocation (e.g. if it is pc relative, absolute, or something else), so the linker can use this information to calculate values that need to be relocated
-  ```relocate``` - The function does what its name says, it writes the given values to addresses in ways specific to the given relocation

But before implementing the beforementioned class, we should probably define target relocations. This is done by adding a def file with target's relocation type names with its values to **llvm/include/llvm/BinaryFormat/ELFRelocs**. Example (RISCV.def):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/reloc-def-riscv2.png?raw=true" />
</div>
<br />

These are only some of relocations from RISC-V, but you can see the point. Include this file in **llvm/include/llvm/BinaryFormat/ELF.h** in an unnamed enum, after the definition of **ELF_RELOC**, but before it is undefined. After this, you have an enum with all your target's reloc types. Here, you can also put processor specific flags. Example (RISCV):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/e-flags-riscv.png?raw=true">
</div>
<br />

Another thing, one should put in **llvm/include/llvm/BinaryFormat/ELF.h** is the target identifier. Example: 

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/supported-machines.png?raw=true">
</div>
<br />

After that, we need to enable the new target by putting a new case in ```TargetInfo```'s ```getTarget``` function. Example:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/get-target.png?raw=true">
</div>
<br />

The specific get target function should be a part of ```TargetInfo``` class and should look something like this:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/get-riscv-target-info.png?raw=true">
</div>
<br />



Now we can start implementing the before mentioned class derived from **TargetInfo** (defined in **lld/ELF/Target.h**). The Target classes are defined in **lld/ELF/Arch** directory. Example of class derived from **TargetInfo** (RISCV):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/target-info-riscv.png?raw=true">
</div>
<br />

As it can be seen on the picture, there are more functions than just ```getRelExpr``` and ```relocate``` that are overriden, they are not needed for the basic functionality of the lld linker itself, so we won't focus on them. But, you'll probably need to implement them later as well.

Implementation of ```getRelExpr``` is rather easy, it is basically a switch case function returning the relocation expressions. Relocation expressions tell the linker how to calculate the values that are needed and there are plenty target agnostic expressions such as **R_ABS**, **R_PC** and **R_GOT_PC** which calculate values absolutely, pc relative or pc relative to got entry. Example of ```getRelExpr``` (RISCV):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/get-rel-expr-riscv.png?raw=true">
</div>
<br />

We can see we return **R_ABS** expression for absolute relocations and **R_RISCV_ADD** relocation for some **RISCV** specific relocations, which we'll touch later.

Implementation of ```relocate``` is a bit more complicated. but it is straight forward. Again, it is a switch case function that depending on the type of the relocation writes the given value to a passed location in a specific way. Example of ```relocate``` (RISCV):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/relocate-riscv.png?raw=true">
</div>
<br />

So depending on the type of the relocation, we alter the calculated value and write it to a passed location. Some relocations check the values passed to see if they fulfill their requirements and abort the linking if they don't.

After this class is implemented, all we need to do is add it to  **lld/ELF/CMakeLists.txt**. Example:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/lld-cmake-lists.png?raw=true">
</div>
<br />

Then we can make our new lld that supports a new architecture for example like this (if in directory of llvm):
  - ```cmake -G "Ninja" -DLLVM_ENABLE_PROJECTS="lld;" -B build ./llvm -DCMAKE_BUILD_TYPE=Debug```
  - ```ninja -C build```

And if no errors occur, you have built your linker.

But not too fast, this isn't the end. Every architecture has specific behaviours and so does every new one we try to implement, and in lld there are many target specific features. We have already seen that **RISCV** has a relocation expression that is a RISCV-only expression (**R_RISCV_ADD**), but there are also target specific symbols, registers, sections, etc. that we have to take care of.

## nanoMIPS target specific features

After implementing the basic ```TargetInfo``` derivation class, we need to see what target specific actions our linker needs. To demonstrate some of them, I'll present what was needed for nanoMIPS architecture beyond the ```getRelExpr``` and ```relocate``` functions. Of course other architectures will differ. Analyzing lld code is a good way of finding out what should be done differently for each architecture, or you can read the documentation for the architecture itself and see if it is somehow implemented in lld.

As **MIPS** architecture **nanoMIPS** also uses a gp register, which is used to access constants, global variables as well as got table. The program itself, at first doesn't know where should gp point to, so it is left to the linker and the linker script to put it on the appropriate address using **_gp** symbol. The symbol is defined in ```addReservedSymbols``` function in **lld/ELF/Writer.cpp**:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/add-gp-nanomips.png?raw=true">
</div>
<br />

Note that the ```ElfSym::mipsGp``` will probably be changed to ```ElfSym::nanoMipsGp``` to avoid confusion, but this is the current implementation. The ```addAbsolute``` just adds a ```Defined``` symbol with value and size 0 to the symbol table.

There is also a need for some different relocation value calculation, so for **nanoMIPS** we have introduced 3 new relocation expressions. These 3 new relocation expressions are inserted in ```RelExpr``` enum in the **lld/ELF/Relocations.h** file:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/rel-expr-nanomips.png?raw=true">
</div>
<br />

Because **R_NANOMIPS_GPREL** and **R_NANOMIPS_PAGE_PC** are relative relocation (**pc**, **gp**, **got**, etc.), we need to add them in ```isRelExpr``` function in the **lld/ELF/Relocations.cpp**:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/is-rel-expr-nanomips.png?raw=true">
</div>
<br />

The **R_NANOMIPS_PAGE_PC** expression is used for **R_NANOMIPS_PC_HI20** relocation, as it is calculated a little bit differently. So for the case of **R_NANOMIPS_PC_HI20** in the ```getRelExpr``` function we return **R_NANOMIPS_PAGE_PC**. And then in ```getRelocTargetVA``` function in **lld/ELF/InputSection.cpp** we have:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/page-pc-va-nanomips.png?raw=true">
</div>
<br />

Where ```getNanoMipsPage``` just extracts the upper 20 bits of the value it is passed. As for the **R_NANOMIPS_GPREL** expression, it is used for gprel relocations of **nanoMIPS**:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/get-rel-expr-gprel-nanomips.png?raw=true">
</div>
<br />

Later on in ```getRelocTargetVA``` function its calculation is basically the result of the subtraction of **gp** from the relocation symbol value. The most interesting one, the **R_NANOMIPS_NEG_COMPOSITE**, is consisted of either 2 or 3 relocations in a row so its relocation must be done differently. The first relocation must be of type **R_NANOMIPS_NEG**, the next can be  **R_NANOMIPS_ASHIFTR_1** and the last one is usually some absolute relocation. **R_NANOMIPS_NEG_COMPOSITE** in ```getRelocTargetVA``` just returns the negative symbol value added with negative addend, but the logic for relocate is now different:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/relocate-alloc-nanomips3.png?raw=true">
</div>
<br />

Also, because of this relocation, we have overriden ```TargetInfo```'s ```relocateAlloc``` function.

Here you can see that relocation of **R_NANOMIPS_NEG_COMPOSITE** expression is different than other expressions, and it calls a special function called ```getNanoMipsNegCompositeRelDataAlloc```, which calculates the value we need to relocate, and later we call the target's ```relocate``` function as we would do in normal circumstances. It should be also noted that the iteration through relocations has been changed in the for loop to using the iterator, so that ```getNanoMipsNegCompositeRelDataAlloc``` can update it. It needs to update it because multiple relocations are processing the needed value and handling them one by one wouldn't be correct, so we need to skip them.

The last target specific feature of **nanoMIPS** that I'll describe is the **ABI** flags section. Named **.nanoMIPS.abiflags** the section is very similar to **.MIPS.abiflags** (structure is the same, but some extension flags differ). This section is unique for the final program and it is used to provide the information to the program loader so it can determine what are the requirements of the application (e.g. hard or soft floating point calculation). The information that is written to it is collected from all the object files that contribute to the final program, as they also have their own **ABI** flags section, which describe the object file's requirements.

Just like **MIPS ABI** flags section, the **nanoMIPS** variant is implemented as a ```SyntheticSection```:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/abi-flags-section-nanomips-h.png?raw=true">
</div>
<br />

First of all, ```Elf_NanoMips_ABIFlags``` is just a struct representing the structure of the **ABI** flags section for **nanoMIPS**. The most important function here is ```create``` and it is invoked by ```get``` if the **ABI** flags section hasn't been created yet. What ```create``` actually does is just iterate through all **nanoMIPS.abiflags** ```InputSection```s that are found during linking of the program and try to synthesize a global **nanoMIPS.abiflags** section that will be put in the output file. ```InputSection```s that are used are marked dead as we no longer need them and they won't influence the output program anymore. What it also does, if an object file doesn't have a **nanoMIPS.abiflags** section, is that it makes it from the **e_flags** of the object file header, and uses those information to merge with others. 

To add this new ```SyntheticSection``` to the whole linking process we insert this code to ```createSyntheticSections``` in **lld/ELF/Writer.cpp**:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/create-synthetic-sections-nanomips.png?raw=true">
</div>
<br />

Basically we create the **ABI** flags section if not created yet, and add it to the input sections in the linking process.

There is one more specific feature that is not mentioned here, ```calcEFlags```, but its implementation is similar to creation of ```nanoMIPS.abiflags```. As this linker progresses, there will be more target specific features implemented.

## Testing

Every tool in the **llvm** system uses the **lit** testing infrastructure, so does **lld**. So, the best way to test your architecture is to write regression tests in the **llvm** infrastructure and as you update your work check if anything has gone wrong. Tests are located in **lld/test/ELF** directory.

## Useful links and sources

1. [LLD - The LLVM Linker](https://lld.llvm.org/)
2. [The ELF, COFF and Wasm Linkers](https://lld.llvm.org/NewLLD.html)
3. [The LLVM Compiler Infrastructure](https://llvm.org/)
4. [Getting Started with the LLVM System](https://llvm.org/docs/GettingStarted.html)
5. [LLVM Testing Infrastructure Guide](https://llvm.org/docs/TestingGuide.html)
6. [lit - LLVM Integrated Tester](https://llvm.org/docs/CommandGuide/lit.html)
7. [llvm-project](https://github.com/llvm/llvm-project)


## Requirements deo




<!-- *italic* **bold** ~~strikethrough~~ [link](https://github.com) ![alt text](https://static.vecteezy.com/system/resources/thumbnails/025/220/125/small_2x/picture-a-captivating-scene-of-a-tranquil-lake-at-sunset-ai-generative-photo.jpg)

- List 1
- List 2
- List 3
  - List 3.1
  - List 3.2
    - List 3.2.1

1. Ordered 1
2. Ordered 2
3. Ordered 3
   1. Ordered 3.1
   2. Ordered 3.2
      1. Oredered 3.2.1

# First tier header
## Second tier header
### Third tier header

---
---

> "Quote, something"

> -Me

```
this is code
``` -->