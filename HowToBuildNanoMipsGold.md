# How to build gold linker for nanoMIPS architecture

## Table of content



## Getting the source

The source code of gold for nanoMIPS is located on https://github.com/MediaTek-Labs/binutils-gdb, so clone it:

```git clone https://github.com/MediaTek-Labs/binutils-gdb.git```


## Building the nanoMIPS gold linker

### In place:

First we must build an assembler to build a linker:

```
cd <binutils-gdb-path>
git checkout nmips/binutils
./configure --disable-werror --target=nanomips-elf && make
cd gas && ./configure --disable-werror --target=nanomips-elf
make install DESTDIR=<absolute-install-path>
```

Now make the linker itself:
```
cd <binutils-gdb-path>
git checkout mtk-pub/gold_v7
./configure --disable-werror --target=nanomips-elf && make
cd gold && ./configure --disable-werror --target=nanomips-elf
make install DESTDIR=<install-path>
```

Note: On every make command, you can add -jN option, to parallelize
the make process on N threads (to speed things up).

To test the build use (in gold directory):
```
make check
```

Note: If you want to make changes to tests, you'll need autoreconf2.64 tool (not that easy to install), and instead of this step:
```
cd gold && ./configure --disable-werror --target=nanomips-elf
```

Use this:
```
cd gold && autoreconf2.64 --install && ./configure --disable-werror --target=nanomips-elf
```

