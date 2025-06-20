## Table of contents
- [Solving issues in nanoMIPS lld](#solving-issues-in-nanomips-lld)
  - [abiflags section problem when rebasing from lld16 to lld20](#abiflags-section-problem-when-rebasing-from-lld16-to-lld20)
    - [Problem](#problem)
    - [Solution](#solution)
    - [New problem](#new-problem)
    - [Solution for the new problem](#solution-for-the-new-problem)

# Solving issues in nanoMIPS lld

Purpose of this document is to show how I solve problems in nanoMIPS lld shown on couple of real examples.

## abiflags section problem when rebasing from lld16 to lld20

While rebasing onto the llvm20 we encountered that several of the lld nanoMIPS (and much others as well)
tests are failing, one of them was the *nanomips-abiflags-section.s* test.

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
assembly files to object files (*nanomips-abiflags-section-sup.s* only contains some dummy code, irrelevant for the test), and that
they have some flags added to them (+soft-float,-tlb,+crc etc.), later on the object files are linked and ```llvm-objdump``` is
used for checking the contents of the **nanoMIPS.abiflags** section. This whole cycle is done for several examples. We have the CHECK-SAME-FLAGS, CHECK-DIFFERENT-FLAGS and CHECK-EFLAGS case:
- CHECK-SAME-FLAGS: both object files have the same flags marked for their compiling (thus same abiflags sections, I'll touch on abiflags section later)
- CHECK-DIFFERENT-FLAGS: the second object file has somewhat different flags specified, and we want to check how this is processed in the linker -> this is the place we get our error from
- CHECK-EFLAGS: Removed **.nanoMIPS.abiflags** section from an object file, to see if we inherit abiflags from eflags

But before we continue, something about the **.nanoMIPS.abiflags** section:
- It is a specific section (mostly same with MIPS' **.MIPS.abiflags** section), which contains what are requirements or what is available in the application. Here we have information like what type of floating point calculation is used, what is the size of a general purpose register and whether some extensions are used (CRC for example).
- It is a section fixed in size (24 bytes), and its contents are calculated by mixing all the input object files' **.nanoMIPS.abiflags** sections
- More info here: http://codescape.mips.com/components/toolchain/nanomips/2019.03-01/docs/MIPS_nanoMIPS_ABI_supplement_01_03_DN00179.pdf in paragraph 5.2

So for some reason, our CHECK-DIFFERENT-FLAGS is failing so the idea is to look at the file we generate by linking these different cases (run the steps from ```llvm-lit``` manually and inspect the output).

When run manually here is what we get from the command that FileCheck checks (```llvm-objdump -s```):

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/manual-test-output.png?raw=true" />
</div>
<br />

So from here we see that even though **.nanoMIPS.abiflags** section is not 24 bytes long, but 48 bytes. And it looks like these are two **.nanoMIPS.abiflags** sections concatenated, which means the algorithm for combining multiple abiflags sections is not working for some reason.

### Solution

So by searching or knowing the code, we get that the ```NanoMipsAbiFlagsSection``` is created as a synthetic section in ```NanoMipsAbiFlagsSection<ELFT>::create```
function. We check if it is called by adding a simple ```llvm::dbgs``` print, and it truly isn't called. Now, other architecture have similar sections,
the most similar one is the MIPS architecture, which also has the abiflags section. This section is added to the input sections
in the ```elf::createSyntheticSections``` function:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/mips-abiflags-addition.png?raw=true" />
</div>
<br />

This kind of code doesn't exist in nanoMIPS yet, so we are going to add it, and hope it solves our problem!

Here is the most significant change and it is in ```elf::createSyntheticSections```:
<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/nanomips-abiflags-addition.png?raw=true" />
</div>
<br />

There are other changes as well, but they are mostly adapting how the abiFlags section was handled before and how it is handled now.

Now when we run the *nanomips-abiflags-section.s* test, it passes, great! But, when we check all of lld tests we see that there are some more nanoMIPS tests that fail now (and they weren't failing before) - *nanomips-abs-relocs.s* for example.

Let's see what is the new problem now.

### New problem

When we run the *nanomips-abs-relocs.s* test, we get this:
<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/new-problem-test-out.png?raw=true" />
</div>
<br />

And this is the relevant part of the test:
<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/abs-reloc-test-part.png?raw=true" />
</div>
<br />


Note: Symbol ```b``` is defined in the other file (*Inputs/nanomips-abs-relocs-sup.s*).

So for some reason our check for symbol b fails, the steps that this test actually does is use llvm-mc to assemble *nanomips-abs-relocs.s* file and *Inputs/nanomips-abs-relocs-sup.s* file, link the with ld.lld and later output the symbol table and program disassembly with llvm-objdump.

Let's run these steps manually and see what we get.

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/abs-relocs-sym-tab-and-disasm.png?raw=true" />
</div>
<br />

Now for some reason, we don't even see the symbol table, why? This is a bit tricky and it seems really unrelated to the abiflags section, so we begin the inspection.


### Solution for the new problem

It seems like the symbol table section doesn't even exist, let's check the section headers.

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/section-headers-abs-relocs-objdump.png?raw=true" />
</div>
<br />

We don't get much info from here, but the suspicious thing is that the **.nanoMIPS.abiflags** and **.symtab** are both on the same VMA, which is weird, they should definitely be on different addresses. Let's try using ```llvm-readelf``` to check for more information:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/section-headers-abs-relocs-readelf.png?raw=true" />
</div>
<br />

Now here we see that the **.nanoMIPS.abiflags** section and **.symtab** section are both treated as **SYMTAB**, but why? We need to investigate where this problem starts and how to solve this.

First thing I checked is whether the MIPS object files get linked properly and the **.MIPS.abiflags** section gets generated properly. First we create an assembly file that both MIPS and nanoMIPS can assemble:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/mips-example.png?raw=true" />
</div>
<br />

Now we run these steps:
```
llvm-mc -filetype=obj -triple mips-elf mips-example.s -o mips-example.o
ld.lld mips-example.s -o mips-example
llvm-objdump -h mips-example
```

We get this:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/mips-example-section-headers.png?raw=true" />
</div>
<br />

So here we see that the **.MIPS.abiflags** has its own **VMA** address, and even is of type **DATA** which is good, while the same compiling and linking process repeated with nanomips yields in a similar output as the abs-relocs one - the symbol table and **.nanoMIPS.abiflags** have same addresses (not that they matter to symbol tables) and are both treated as symbol tables. This means everything with MIPS is okay, and that the problem is nanoMIPS related.

Now one thing that I forgot to mention, while reading about [abiflags](http://codescape.mips.com/components/toolchain/nanomips/2019.03-01/docs/MIPS_nanoMIPS_ABI_supplement_01_03_DN00179.pdf) is that this section is put in its own program header. We can check this on MIPS and nanoMIPS version wtih ```llvm-objdump```.

MIPS version:
<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/mips-example-program-headers.png?raw=true" />
</div>
<br />

The first **UNKNOWN** program header corresponds to the **.MIPS.abiflags** section (we check that by the addresses that are in this region's range).

nanoMIPS version:
<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/nanomips-example-program-headers.png?raw=true" />
</div>
<br />

We see, in nanoMIPS version, there is no such a program header that corresponds to the **.nanoMIPS.abiflags** section. We'll start by searching in lld why this happens. Okay, by searching a little bit where we add program headers we found that ```Writer<ELFT>::finalizeSections``` adds a **PT_NANOMIPS_ABIGLAGS** program header (search was conducted by searching files for abiflags). So we add some prints to see whether we actually add this program header:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/putting-nanoMIPS-program-headers.png?raw=true" />
</div>
<br />

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/really-putting-nanoMIPS-program-headers.png?raw=true" />
</div>
<br />

The first picture is a print prior to the function where we actually add the program header we need, and the second one is to see if all conditions are met to add it. When linking the simple example program with this we get this output:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/nanomips-example-link-output-1.png?raw=true" />
</div>
<br />

We see that the we haven't printed the "Really putting them nanoMIPS phdrs", and that is because we 
haven't got any sections of the shType (**SHT_NANOMIPS_ABIFLAGS**) that we speicified for this program
header, which again doesn't seem that possible. No we'll try by listing the output sections and their
types to see what is happening?

Luckily for us, addition of output section is also done in ```Writer<ELFT>::finalizeSections```. Add a print here to see what we are getting:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/output-section-addition.png?raw=true" />
</div>
<br />

Now let's see the output for the example project.

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/output-section-addition-output.png?raw=true" />
</div>
<br />

From here, we see that abiflags section is of type 2 (**SHT_SYMTAB**) instead of 0x70000000 (**SHT_NANOMIPS_ABIFLAGS**). Okay, so there is something probably wrong with the generation of the .nanoMIPS.abiflags section. Now the solution is pretty straight forward and I could've got there way more easily, but I am going to show the way I did it.

So the first idea that comes to my mind, is check the llvm-mc and llvm-objdump, if they are working properly? Again, this could be done without prints with using elfread alongside objdump, but anyways this is the way that is shown here.

Put some prints in the function that emits abiflags section for llvm-mc and see what we get. Also, check whether we get that same section header type from the objdump output.

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/llvm-mc-emit-abiflags.png?raw=true" />
</div>
<br />

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/objdump-is-section-data.png?raw=true" />
</div>
<br />

llvm-mc output:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/llvm-mc-output.png?raw=true" />
</div>
<br />

Here we see that everything is emitted and that the type is fine (1879048192 == 0x70000000), so we assume that llvm-mc is fine.

llvm-objdump output:
<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/objdump-output.png?raw=true" />
</div>
<br />

llvm-objdump also shows us now that the llvm-mc did a good job, and that everything is probably the linker's fault. So we continue where we got with the linker, and that was adding the outputSections to their final list. So we can see from 5 pictures above that the output sections are added from ```ctx.script->sectionCommands```, let's try to investigate this (although we don't have a linker script, this structure is still used to place other remaining sections). We'll try to find out when this structure is updated - by inspecting it during the ```LinkDriver::link``` function:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/ctx-script-print-1.png?raw=true" />
</div>
<br />

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/ctx-script-print-2.png?raw=true" />
</div>
<br />

And this is what we get from running this, seems like all output sections are added in ```addOrphanSections```. Let's see what we got:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/list-orphan-sections.png?raw=true" />
</div>
<br />

Output:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/list-orphan-sections-output.png?raw=true" />
</div>
<br />

*\<internal\>* means that the section is created during the linking, doesn't come from an object file. We see that there are two **.nanoMIPS.abiflags** output sections created, one corresponds to the one from the nanomips-example.o file and it is not live - which is okay because during creation of the global **.nanoMIPS.abiflags** section, we mark all of the input ones as dead. Here we also see that the global one (marked with *\<internal\>*) is not properly created, let's dig into that.

To short this a bit, we'll just talk through further in the process. So we go deeper by putting prints in the ```addOrphanSections()``` function to see how are adding those output section descriptions, this leads as to the ```addInputSec``` function, which then uses ```createSection``` adn so on, from here we didn't find anything new the section is created badly. So we get back to the creation of the **.nanoMIPS.abiflags** section, and with some tweaking of the *type* field we see that the problem can be fixed like that:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/temp-fix.png?raw=true" />
</div>
<br />

So this indicates to us, that there might be a problem in the constructor, which there really is. Arguments of the constructor were wrnogly placed the type and flags arguments were misplaced:

<div align="center">
  <img src="https://github.com/AndrijaSyrmia/Docs/blob/master/assets/solving-issues-in-nanomips-lld/misplaced-constructor.png?raw=true" />
</div>
<br />

By just changing the positions of those two arguments, we get the proper behaviour (simple solution, and it could've been way easier to solve this, but here were the steps). Later on we check this with testing the regression tests again, and see that we have truly fixed this issue.

That's all for this issue.