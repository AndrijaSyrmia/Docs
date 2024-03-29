# Adding a new ELF Machine Target to LLD

In this blog I'll cover a method to add new ELF Machine Targets to lld, then I'll talk about implementing target specific relaxations (or transformations) and lastly comes a brief walkthrough of nanoMIPS target implementation that I've made.

## Why LLD

So the first question that comes to me, is why should we use lld when we have gnu linkers (bfd and gold) that are working fine. What people like to emphasize the most is lld's speed. On average lld runs twice as fast as the GNU gold linker, but it tends to use more memory. It is small, it's source code is several times smaller in term of code line count. But what I find to be most helpful for developers are more flexible data structures and algorithms then on gnu's gold linker. The code is also easier to understand then in gold, so it is easier for developers to work on it further. Lastly, it works good with clang as clang can generate bitcode object files which can be then sent to lld to perform link time optimizations on whole programs.

I hope that this explains the whys behind lld's popularity.

## Useful structures in LLD

In this section, I'll briefly describe structures in lld that were useful to me, when it comes to implementing a new Machine Target to LLD.
- ```InputSection``` - represents a section from an object file or a synthetic one that should be somehow put in the output file or discarded. One can easily traverse its relocations and access its data.
- ```OutputSection``` - represents a section in an output file containing ```InputSection```s from various sources.
- ```Symbol``` - represents a symbol from a object file, archive or linker defined symbols.
- ```Relocation``` - represents a relocation in an object file, with its type, expression, offset in ```InputSection``` and ```Symbol``` to which the ```Relocation``` refers
- ```Writer``` - A class that mainly writes the data from ```OutputSection``s to the output file
- ```Driver``` - Main driver of the lld linker, it is responsible for controlling the linker's flow of work
- ```Options.td``` - Not a structure itself, but it defines options that can be passed to the linker, most of the time those options are somehow stored in ```Config``` structure

## Adding a new ELF Machine Target

Simply said, all you need to do to add a new machine target to lld is implement a class representing your architecture as a derrivation of lld's TargetInfo class. Easier said than done. To achieve this you need to implement two functions:
-  ```getRelExpr``` - Used to return the sort of relocation (e.g. if it is pc relative, absolute, or something else), so the linker can use this information to calculate values that need to be relocated
-  ```relocate``` - The function does what its name says, it writes the given values to addresses in ways specific to the given Relocation

But before implementing the beforementioned class, we should probably define target relocations. This is done by adding a def file with target's relocation type names with its values to **llvm/include/llvm/BinaryFormat/ELFRelocs**. Example:






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