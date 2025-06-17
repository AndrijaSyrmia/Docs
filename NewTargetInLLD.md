## Table of content

- [Adding a new ELF Machine Target to LLD](#adding-a-new-elf-machine-target-to-lld)
  - [Why LLD](#why-lld)
  - [Useful structures in LLD](#useful-structures-in-lld)
  - [LLD linking steps](#lld-linking-steps)
  - [Adding a new ELF Machine Target](#adding-a-new-elf-machine-target)
  - [nanoMIPS target specific features](#nanomips-target-specific-features)
  - [Linker relaxations](#linker-relaxations)
  - [nanoMIPS linker transformations](#nanomips-linker-transformations)
  - [Testing](#testing)
  - [Conclusion](#conclusion)
  - [Useful links and sources](#useful-links-and-sources)
  - [Requirements](#requirements)

# Adding a new ELF Machine Target to LLD

In this blog we'll cover a method to add new **ELF Machine Targets** to **lld**, then we'll talk about implementing target specific relaxations (or transformations). Throughout the blog the implementation of **nanoMIPS** target will be showcased as well. The examples shown are done on llvm-16 (lld-16).

## Why LLD

So the first question that comes to me, is why should we use lld when we have gnu linkers (bfd and gold) that are working fine. What people like to emphasize the most is lld's speed. On average lld runs twice as fast as the GNU gold linker, but it tends to use more memory. It is small, it's source code is several times smaller in terms of code line count. But what I find to be most helpful for developers are more flexible data structures and algorithms then on gnu's gold linker. The code is also easier to understand then in gold, so it is easier for developers to work on it further. Lastly, it works good with clang as clang can generate bitcode object files which can be then sent to lld to perform link time optimizations on whole programs. [[1]](#useful-links-and-sources) [[2]](#useful-links-and-sources)

I hope that this explains the whys behind lld's popularity.

## Useful structures in LLD

In this section, I'll briefly describe structures in lld that were useful to me, when it comes to implementing a new ELF Machine Target. [[3]](#useful-links-and-sources)
- ```InputSection``` - represents a section from an object file or a synthetic one that should be somehow put in the output file or discarded. One can easily traverse its relocations and access its data.
- ```OutputSection``` - represents a section in an output file containing ```InputSection```s from various sources.
- ```Symbol``` - represents a symbol from an object file, archive or linker defined symbols.
- ```Relocation``` - represents a relocation in an ```InputSection```, with its type, expression, offset in ```InputSection``` and ```Symbol``` to which the ```Relocation``` refers
- ```Writer``` - A class that mainly writes the data from ```OutputSection```s to the output file
- ```Driver``` - Main driver of the lld linker, it is responsible for controlling the linker's flow of work
- ```Options.td``` - Not a structure itself, but it defines options that can be passed to the linker, most of the time those option values are stored in ```Config``` structure

## LLD linking steps

Here, we'll quickly walk through steps of linking with **lld**. 
1. First **lld** tries to deduce which linker should be used (**elf** linking, **coff** linking, **macho** linking or **wasm** linking), it is usually deduced by calling a specific linker instead of just **lld** (**ld.lld** is the linker for **Unix**) or by passing the -flavor option (**gnu** if the same outcome as **ld.lld** is expected).
2. We are talking about **elf** linking from now on, so after we've determined that we want to use the **elf** linker, initialization of needed structures occurs (configuration, script, context, etc.) and the ```linkerMain``` is called.
3. In ```linkerMain``` arguments passed to the linker are parsed, and most commonly somehow remembered in the ```Config``` structure. Also, linker tries to find all the needed files, libraries and creates their structures before ```link``` is called.
4. After processing some specific arguments (if passed), files and libraries passed to the linker are parsed, creating needed ```InputSection```s and filling the symbol table in ```parseFile``` and ```initSectionsAndLocalSyms```. When parsing is completed for all the files, then we check for duplicate symbols in the ```postParseObjectFile``` function. After no new symbols are there to be added (except of some synthetic ones), some target dependent properties are set, such as ```eflags```, ```maxPageSize```, etc. Then, sections are copied into partitions (those that we want in the output file) and synthetic sections are created. Next, the linker script is read for section commands and through ```finalizeInputSections``` function of ```OutputSection``` ```InputSection```s are added to the corresponding ```OutputSection```s. Finally ```writeResult``` is called.
5. ```writeResult``` runs the ```Writer```. Firstly, ```finalizeSections``` is invoked. This function scans relocations, adds symbols to the final symbol table, creates a list of ```OutputSection```s, finalizes synthetic sections, assigns final value to some address dependent symbols, changes data in some sections (**relaxations**, **thunks**) and fills out section headers. At the end ```writeHeaders``` and ```writeSections``` take place and write the program to the output file, resolving relocations also happens in this phase.

## Adding a new ELF Machine Target

Simply said, all you need to do to add a new machine target to lld is implement a class representing your architecture as a derivation of lld's ```TargetInfo``` class. Easier said than done. To achieve this you need to implement two functions:
-  ```getRelExpr``` - Used to return the sort of relocation (e.g. if it is pc relative, absolute, or something else), so the linker can use this information to calculate values that need to be relocated
-  ```relocate``` - The function does what its name says, it writes the given values to addresses in ways specific to the given relocation

But before implementing the aforementioned class, we should probably define target relocations. This is done by adding a def file with target's relocation type names with its values to **llvm/include/llvm/BinaryFormat/ELFRelocs**. Example (**RISCV.def**):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/reloc-def-riscv2.png?raw=true" />
</div>
<br />

These are only some of relocations from **RISC-V**, but you can see the point. Include this file in **llvm/include/llvm/BinaryFormat/ELF.h** in an unnamed enum, after the definition of **ELF_RELOC**, but before it is undefined. After this, you have an enum with all your target's reloc types. Here, you can also put processor specific flags. Example (**RISCV**):

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



Now we can start implementing the aforementioned class derived from **TargetInfo** (defined in **lld/ELF/Target.h**). The Target classes are defined in **lld/ELF/Arch** directory. Example of class derived from **TargetInfo** (**RISCV**):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/target-info-riscv.png?raw=true">
</div>
<br />

As it can be seen on the picture, there are more functions than just ```getRelExpr``` and ```relocate``` that are overriden, they are not needed for the basic functionality of the lld linker itself, so we won't focus on them. But, you'll probably need to implement them later as well.

Implementation of ```getRelExpr``` is rather easy, it is basically a switch case function returning the relocation expressions. Relocation expressions tell the linker how to calculate the values that are needed and there are plenty target agnostic expressions such as **R_ABS**, **R_PC** and **R_GOT_PC** which calculate values absolutely, pc relative or pc relative to got entry. Example of ```getRelExpr``` (**RISCV**):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/get-rel-expr-riscv.png?raw=true">
</div>
<br />

We can see we return **R_ABS** expression for absolute relocations and **R_RISCV_ADD** relocation for some **RISCV** specific relocations.

Implementation of ```relocate``` is a bit more complicated. but it is straight forward. Again, it is a switch case function that depending on the type of the relocation writes the given value to a passed location in a specific way. Example of ```relocate``` (**RISCV**):

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
  - ```cmake -G "Ninja" -DLLVM_ENABLE_PROJECTS="lld;" -B build ./llvm```
  - ```ninja -C build```

And if no errors occur, you have built your linker.

But not too fast, this isn't the end. Every architecture has specific behaviours and so does every new one we try to implement, and in lld there are many target specific features. We have already seen that **RISCV** has a relocation expression that is a **RISCV**-only expression (**R_RISCV_ADD**), but there are also target specific symbols, registers, sections, etc. that we have to take care of.

## nanoMIPS target specific features

After implementing the basic ```TargetInfo``` derivation class, we need to see what target specific actions our linker needs. To demonstrate some of them, I'll present what was needed for **nanoMIPS** architecture beyond the ```getRelExpr``` and ```relocate``` functions. Of course, other architectures will differ. Analyzing lld code is a good way of finding out what should be done differently for each architecture, alongside reading the documentation for the architecture itself and see if it is somehow implemented in lld.

As **MIPS** architecture **nanoMIPS** also uses a **gp** register, which is used to access constants, global variables as well as **got** table. The program itself, at first doesn't know where should **gp** point to, so it is left to the linker and the linker script to put it on the appropriate address using **_gp** symbol. The symbol is defined in ```addReservedSymbols``` function in **lld/ELF/Writer.cpp**:

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

Later on in ```getRelocTargetVA``` function its calculation is basically the result of the subtraction of **gp** from the relocation symbol value. The most interesting one, the **R_NANOMIPS_NEG_COMPOSITE**, is consisted of multiple relocations in a row so its relocation must be done differently. The first relocation must be of type **R_NANOMIPS_NEG**, the next ones are usually absolute relocations. **R_NANOMIPS_NEG_COMPOSITE** in ```getRelocTargetVA``` just returns the negative symbol value added with negative addend, but the logic for relocate is now different:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/relocate-alloc-composite.png?raw=true">
</div>
<br />

Also, because of this relocation, we have overriden ```TargetInfo```'s ```relocateAlloc``` function.

Here you can see that relocation of **R_NANOMIPS_NEG_COMPOSITE** (it is the main composite relocation) expression is different than other expressions. It calculates the value of all relocations that are on the same section offset and combines them to one value (**R_NANOMIPS_NEG_COMPOSITE** is usually used for expressions like ```a - b``` or even ```a - b << 1```).

The last target specific feature of **nanoMIPS** that I'll describe is the **ABI** flags section. Named **.nanoMIPS.abiflags** the section is very similar to **.MIPS.abiflags** (structure is the same, but some extension flags differ). This section is unique for the final program and it is used to provide the information to the program loader so it can determine what are the requirements of the application (e.g. hard or soft floating point calculation). The information that is written to it is collected from all the object files that contribute to the final program, as they also have their own **ABI** flags section, which describe the object file's requirements.

Just like **MIPS ABI** flags section, the **nanoMIPS** variant is implemented as a ```SyntheticSection```:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/abi-flags-section-nanomips-h.png?raw=true">
</div>
<br />

First of all, ```Elf_NanoMips_ABIFlags``` is just a struct representing the structure of the **ABI** flags section for **nanoMIPS** [[8]](#useful-links-and-sources). The most important function here is ```create``` and it is invoked by ```get``` if the **ABI** flags section hasn't been created yet. What ```create``` actually does is just iterate through all **nanoMIPS.abiflags** ```InputSection```s that are found during linking of the program and try to synthesize a global **nanoMIPS.abiflags** section that will be put in the output file. ```InputSection```s that are used are marked dead as we no longer need them and they won't influence the output program anymore. What it also does, if an object file doesn't have a **nanoMIPS.abiflags** section, is that it makes it from the **e_flags** of the object file header, and uses those information to merge with others. 

To add this new ```SyntheticSection``` to the whole linking process we insert this code to ```createSyntheticSections``` in **lld/ELF/Writer.cpp**:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/create-synthetic-sections-nanomips.png?raw=true">
</div>
<br />

Basically we create the **ABI** flags section if not created yet, and add it to the input sections in the linking process.

There is one more specific feature that is not mentioned here, ```calcEFlags```, but its implementation is similar to creation of ```nanoMIPS.abiflags```. As this linker progresses, there will be more target specific features implemented.

## Linker relaxations

Just like there is a bunch of various compiler optimizations, so there are some linker ones. One of these optimizations are called relaxations. Their goal is to scan the output program and see if some instructions can be compacted (e.g. 32bit instruction to 16bit one), or if some instruction groups can be replaced by other shorter and/or faster groups (e.g. load the highest 20 bits, then add the lowest 12, can be replaced by just loading the whole value if the proper conditions are met). This cannot be done during compilation, as the compiler doesn't know the layout of the program, nor it knows what are the values of all the symbols and where they originate from.

In lld there is a couple of ways to achieve this. The first one is by using relocation expressions (custom or already present ones). An example of these kind of relaxations would be relaxations of **TLS** [[9]](#useful-links-and-sources) access models. Shortly, **TLS** has several access models, **General Dynamic**, **Local Dynamic**, **Initial Executable** and **Local Executable**, where **Global Dynamic** is the least optimized one and the **Local Executable** is the most optimized one. **lld** itself can already determine whether access models can be relaxed in the ```handleTlsRelocation``` function in **lld/ELF/Relocation.cpp** (you'll maybe need to add something target specific to this function, if you are adding a new machine target). Here is one snippet of the code:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/handle-gd-tls-relocation.png?raw=true">
</div>
<br />

You can see here, that we add relocations with relax expressions if possible, that are later processed in ```relocateAlloc``` function. Example (**AArch64**):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/relax-tls-relocate-alloc-aarch64.png?raw=true">
</div>
<br />

Basically, what these ```relaxTls*``` functions do, is just rewrite instructions to their relaxed version.

Another way to relax is to implement a custom relaxer that will do the transformations. Example (**Aarch64**):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/aarch64-relaxer.png?raw=true">
</div>
<br />

This relaxer is used during resolving relocations to try to relax some sequences of instructions. It checks if all the information in associated relocations are valid, and relaxes if the value to be relocated fits the given range. As it is used during resolving of the relocations, it is placed in ```relocateAlloc```:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/aarch64-relaxer-usage.png?raw=true">
</div>
<br />

If it is not possible to relax given instructions, then we normally resolve the relocation.

The last relaxation method that I'll describe (and which **nanoMIPS** uses) is by implementing ```TargetInfo```'s ```relaxOnce``` function. It was initially used by the **RISC-V** [[10]](#useful-links-and-sources) architecture. It needs to be noted that, by the current implementation, target may not need "thunks" and perform relaxations this way ("thunks" are small pieces of code inserted by the linker to extend the range of jump/branch instructions). The function itself is called during finalizing of sections (which is prior to resolving relocations) in ```finalizeAddressDependentContent``` of the **lld/ELF/Writer.cpp** file:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/finalize-address-relax-once.png?raw=true">
</div>
<br />

What this does actually is it runs relaxations on all sections once, and does so iteratively until there is no more changes, as relaxations can later lead to even more relaxations due to code size change. Later in the same function, there is a call to ```LinkerScript```'s  ```assignAddresses``` function, which recalculates section and symbol offsets that are assigned in the script itself. More about this relaxation type comes in the next section.

Of course, there is more ways than the mentioned ones you can use to relax your code, but these are the ones that I've ran into in **lld**.

## nanoMIPS linker transformations

Before we dig into **nanoMIPS** relaxations it should be noted that in **nanoMIPS** there exists an opposite transformation, called expansions. Expansions are necessary for **nanoMIPS**, as sometimes the instructions generated by the compiler cannot reach their targets, and the expansions are used to replace these instructions with those which can.

Now, let's dig in the transformations from where we've finished the previous section - the ```relaxOnce``` function:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/relax-once-nanomips.png?raw=true">
</div>
<br />

Before transforming anything, we need to do some checks:
- what is the current transform state if it is **None** (neither **Relax** nor **Expand**) finish early
- if there are program arguments provided supporting transformations (**--relax**, **--expand**)
- if the output file is not relocatable
- we only transform output sections that contain instructions
- input sections are transformed only if the object file they originate from contains **linkrelax** flag, and if they have at least one relocation

You see here there is something called ```currentTransformation``` which holds info and methods for transforming in particular ways, there are three states of transformations - **Relax**, **Expand**, **None**. This is used to determine in what way will the transformations be performed (**Relax** - for relaxations, **Expand** - for expansions and **None** - no transformations).

Besides these steps, we also initialize auxiliary info (extra data associated with input sections that makes transformations easier) prior to the first transformation. Then we iterate through all output sections and their input sections and try to apply transformations on them through ```transform``` method.

After that we change the state of the current transformation (state will change if the current pass didn't meddle with the code, or after some number of passes) to prepare for the next pass. The **None** state will come after both **Relax** and **Expand** have no changes, or after **Relax** has reached its pass limit and **Expand** has no changes. **Expand** usually comes after the **Relax** state has no changes, and if **Expand** had any changes we go back to **Relax**. And finally, we finalize what was done, basically just doing some final cleanups and restoring what we've changed in ```initTransformAuxInfo```.

Now we'll check the ```transform``` function. This function iterates through relocations of the given ```InputSection```, checks whether it can perform the given transformation (**relaxation** or **expansion**) and performs it if possible.

We start by checking some specific relocations:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/transform-begin-nanomips.png?raw=true">
</div>
<br />

The logic for the first two specific relocations is simple, if we run to **R_NANOMIPS_NORELAX** we shouldn't transform relocations after this one unless we run to **R_NANOMIPS_RELAX** which permits transformation of relocations after it. One more special relocation is **R_NANOMIPS_ALIGN**, it requires different logic than the normal transformations. We'll go back to ```align``` afterwards. 

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/transform-reloc-property-nanomips.png?raw=true">
</div>
<br />

Later on we calculate the value we are relocating, as well as the ```NanoMipsRelocProperty``` for this relocation. ```NanoMipsRelocProperty``` contains size of the instruction types that it is referring to, as well as the bit mask of those types. ```NanoMipsRelocPropertyTable``` is created with **table gen** in **lld/ELF/Arch/NanoMipsRelocProperty.td** which uses **llvm/utils/TableGen/RelocInsPropertyEmitter.cpp** (a custom table gen backend to emit these properties) to do so. Other structures such as ```NanoMipsInsProperty```, ```NanoMipsTransformTemplate``` and ```NanoMipsInsTemplate``` use the same **table gen** file.

Once we've seen that there are transformations for the current relocation, we need to check if there are transformations for the relocation and the instruction it refers to, thus come next steps:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/transform-ins-property.png?raw=true">
</div>
<br />

Here, we read the instruction from the section, and check if there is a ```NanoMipsInsProperty``` for the calculated instruction mask and relocation. If there is we may continue with our transformation. ```NanoMipsInsProperty``` object contains the transformations available for this instruction, as well as helper functions to validate, convert and get registers from the instruction, which are needed during transformations to check and insert new instructions. Later, we check if the length of the instrucion is forced, if so we cannot do transformations with it. The length is forced if there is one of these relocations attached to the instruction: **R_NANOMIPS_FIXED**, **R_NANOMIPS_INSN32** or **R_NANOMIPS_INSN16**.

Further down the transform function, we get the desired transformation (relaxation or expansion):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/transform-get-transform-template-nanomips.png?raw=true">
</div>
<br />

We get the template for the transformation via ```getTransformTemplate``` method. This method checks the relocation, instruction and the value for relocation, alongside some program parameters (e.g. whether we are making a **pcrel** output program) and tries to find the appropriate transformation if possible (**relaxations**) or needed (**expansions**). ```NanoMipsTransformTemplate``` contains instructions needed for the transformation, as well as relocations that go with those instructions (if there is no relocation necessary, **R_NANOMIPS_NONE** is put in this structure).

Finally, if we have the desired transformation, we make changes to the section content:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/transform-change-content-nanomips.png?raw=true">
</div>
<br />

Firstly, we change the section size to fit new instructions via ```updateSectionContent```. This function also updates relocation offsets, and symbol values and sizes that should be changed. After this, we call the ```transform``` method of the ```currentTransformation``` so we can get the instructions we need to write. Instructions are created from the ```NanoMipsTransformTemplate```'s instructions, ```NanoMipsInsProperty```'s functions for extracting and converting registers currently present in that instruction and ```Relocation```'s type. We also, add and change relocations of the processed input section here, to match the new instructions. At the end, we get and write newly generated instructions to the section.

Now when we know how transformations are done, we go back to the aligning:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/align-begin-nanomips.png?raw=true">
</div>
<br />

First, we initialize the needed data:
- **nop** instructions, for filling out the alignment requirements (if not defined differently)
- alignment value
- old and new padding
- default fill value, its fill size and max bytes that can be used for padding

Afterwards, we check if fill size, fill value or maximum number of bytes to fill should be changed:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/align-max-fill-nanomips.png?raw=true">
</div>
<br />

They should be changed if there are **R_NANOMIPS_FILL** or **R_NANOMIPS_MAX** relocs on the same offset as **R_NANOMIPS_ALIGN** that we are processing.

From here we update the content if there were changes:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/align-update-nanomips.png?raw=true">
</div>
<br />

What is interesting here, is that it might be possible to cut a **nop32** instruction on half, so we do a check for that and if it would happen we replace it with 2 **nop16** instructions. After updating we adjust the symbol size of the **R_NANOMIPS_ALIGN** relocation, as it is used to store current padding. After this we write the padding like this:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/align-write-nanomips.png?raw=true">
</div>
<br />

## Testing

Every tool in the **llvm** system uses the **lit** testing infrastructure, so does **lld**. The best way to test your architecture is to write regression tests in the **llvm** infrastructure and as you update your work check if anything has gone wrong. Tests are located in **lld/test/ELF** directory. [[11]](#useful-links-and-sources)[[12]](#useful-links-and-sources)

## Conclusion

In this article, we've first covered basics of the **lld ELF** linker. Then we saw how to implement a new machine target in this linker, what classes need to be implemented with their abstract functions. Later on, we listed some target specific content for **nanoMIPS**, as during implementation of any new machine target we'll definitely have to implement many target specific features. Finally, we dug into the linker relaxations, visiting a few ways to do them in **lld**, as well as explained **nanoMIPS** linker transformations. 


## Useful links and sources

1. [LLD - The LLVM Linker](https://lld.llvm.org/)
2. [LLVM Link Time Optimization: Design and Implementation](https://llvm.org/docs/LinkTimeOptimization.html)
3. [The ELF, COFF and Wasm Linkers](https://lld.llvm.org/NewLLD.html)
4. [The LLVM Compiler Infrastructure](https://llvm.org/)
5. [Getting Started with the LLVM System](https://llvm.org/docs/GettingStarted.html)
6. [llvm-project](https://github.com/llvm/llvm-project)
7. [llvm-16 with nanoMIPS target implementation in lld](https://github.com/AndrijaSyrmia/llvm-project/tree/nanomips-lld-16-new-relaxations)
8. [ELF ABI Supplement for nanoMIPS](http://codescape.mips.com/components/toolchain/nanomips/2019.03-01/docs/MIPS_nanoMIPS_ABI_supplement_01_03_DN00179.pdf)
9. [Thread-Local Storage Access Models](https://docs.oracle.com/cd/E19683-01/817-3677/chapter8-20/index.html)
10. [RISC-V linker relaxation in lld](https://maskray.me/blog/2022-07-10-riscv-linker-relaxation-in-lld)
11. [LLVM Testing Infrastructure Guide](https://llvm.org/docs/TestingGuide.html)
12. [lit - LLVM Integrated Tester](https://llvm.org/docs/CommandGuide/lit.html)

## Requirements

- Implementation of the **nanoMIPS** target is done on llvm version 16.