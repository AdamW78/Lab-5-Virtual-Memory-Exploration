# Lab 5 - Virtual Memory Exploration
> This lab requires a **64-bit Linux environment** with 4096 byte pages, like on portal.

Changelog:

*   2 Oct 2024: remove things left over from last semester.

Possibly working with a partner —

0.  Download the skeleton code ([here](files/vm-explore.tar) or `wget https://www.cs.virginia.edu/~cr4bd/3130/F2024/labhw/files/vm-explore.tar`) and run `make` to compile it.
    
    Run the code like `./lab 0` and observe the output, which is explained below in the [the ./lab program](#program) section below
    
1.  Make a global array like:
    
        alignas(4096) volatile char global_array[4096 * 32];
    
    See the section [array declaration](#array-decl) for an explanation of what this code does.
    
    Change the code in `lab.c`’s `labStuff` so that when the parameter is `1`, it runs some code to write[1](#fn1) to the array.
    
    Make sure you can observe (running `./lab 1`) that when you:
    
    *   writing different pages (4096-byte regions) of the array, that increases the number of faults and the resident set size
    *   writing the same page multiple times, that does not increase the number of faults or the resident set size more than accessing a page once
    
    See the section [Linux and allocate-on-demand](#demand) below for why this should happen.
    
2.  Change the code in `lab.c`’s `labStuff` so that when the parameter is `2`, it runs code that:
    
    *   allocates 1MB of memory, and
    *   accesses a few bytes of the allocation
    
    Make sure you observe (running `./lab 2`) that:
    
    *   the allocation increase the virtual memory size reported is about 1MB, but not the resident set size; and
    *   the resident set size and fault count increases in proportion to how many pages of the allocated memory you access (as in item 1)
3.  Change the code in `lab.c`’s `labStuff` so that when the parameter is `3`, it runs code that increases the virtual memory size by 1048576 bytes and the resident set size by 131072 bytes and performs the _fewest memory accesses possible to do so_.
    
    Hint: You should only need to access each allocated page once.
    
4.  For this and the following item, run `./lab` with [address randomization](#addr-random) disabled using `setarch --addr-no-randomize ./lab ...` (instead of `./lab ...`).
    
    Determine the location of the heap, from `./lab`’s output.
    
    Modify `labStuff` so that when the parameter is `4`, it allocates and _uses_ (reads or writes) 4096 bytes of memory at an address 0x200000 bytes after the end of the heap (rounded up, if necessary, to the next address that is a multiple of 4096 (= 0x1000)).
    
    See [below](#item-45) for information about how to do this.
    
    If successful, you should observe that:
    
    *   the allocation should be visible on the memory layout output by the `./lab` program at the end
        
    *   this results in a small increase in the space used for page table entries, which you may not have seen on the previous items
        
5.  Do the same as item 4, but make it so when `5` it allocates and uses 4096 bytes of memory 0x10000000000 bytes after the end of the heap (again, with the address rounded up to a multiple of 4096 if necessary).
    
    If successful, you should observe that this allocation results in a larger increase in the space used for page table entries than what you did for item 4.
    
6.  Show a TA your labStuff function and the `./lab` program operating for items 1-5 above **or whatever you managed to complete in the lab time**, or submit your `lab.c` (making sure it indicates in a comment anyone you worked with) to the submission site.
    

1 the `./lab` program
=====================

We supply `./lab` program, built from `lab.c` (which you will modify) and `util.c` (which you shouldn’t need to modify).

To run it, run

    ./lab NUMBER

where NUMBER is an integer that will be passed to the function `labStuff` which you will edit later in the lab. You can try `./lab 0` to see an example of the output.

It displays the program’s memory layout, then displays several statistics about the program’s memory usage, then runs the function `labStuff`, then prints the new memory usage and the new memory layout.

When displaying memory layout, each line has a range of hexadecimal addresses and either what file is loaded into that memory or some other label for the memory region. Inaccessible parts of memory are not listed. Since Linux does allocate-on-demand, the memory layout does not reflect what pages are actually allocated to the process — just what the process will be able to successfully access.

When displaying memory usage, the program displays

*   the number of page faults that occured. (Page faults are the hardware exception that occurs when a program tries to access a memory location which has no valid mapping in its page table or which has a valid mapping but without appropriate permission bits set.)
    
    The page faults are divided into minor faults, which are resolved without I/O (such as by allocating a new, empty page or copying a read-only page for copy-on-write), and major faults which require I/O (such as by reading a file from disk into RAM).
    
*   the virtual memory size, the size of available data and code, regardless of whether it is currently mapped in the page table or will be loaded on demand
    
*   the resident set size, which is essentially the size of loaded data and code
    
*   the page table entries, which is the total space used to store page table entries for the program.
    
*   the shared size, the amount of available data/code that may be shared with other programs
    
*   the swap size, the amount of available data/code that would normally be stored in memory, but is temporarily being stored on disk
    

Your tasks in this lab will be to modify the program to change the values displayed for each of these by writing code in the function `labStuff`.

To help compare the memory usage before and after, the program will display the change in parenthesis next to the after values. (There is no similar display for changes in the memory layout; you’ll need to compare in some other way.)

In the exercises in this lab, you’ll modify `labStuff` to do some operations with memory and observe how they change those statistics.

2 Item-specific info/hints/advice
=================================

2.1 Item 1
----------

### 2.1.1 array declaration

The array declaration

    alignas(4096) volatile char global_array[4096 * 32];

declares an array of `4096 * 32` characters.

Declaring it as `volatile` will make our C compilers not optimize away memory accesses to the array[2](#fn2).

Using `alignas(4096)` asks the compiler to place `global_array` at an address that is a multiple of 4096; this will ensure that the array does not share virtual pages with other global variables. It will also ensure that the array starts at the beginning of a page, so array indices 0 through 4095 inclusive will be one virtual page, indices 4096 through 8191 another, and so on. (One needs to include `<stdalign.h>` to use `alignas`, which our template code does.)

We deliberately do not set an initial value for this array to ensure that the memory for it will be constructed from scratch rather than copied from (and potentially shared with, using copy-on-write) an executable file’s data (which may already be in memory before the executable even starts).

### 2.1.2 Linux and allocate-on-demand

On Linux, the kernel usually sets up page table entries only at the last possible moment. That is, when a program first attempts to access a page, it will not yet be in the page table, then an exception will occur and Linux will update the page table entries accordingly.

2.2 Item 2
----------

### 2.2.1 preventing access from being optimized away

Normally when you allocate memory with the stdlib.h functions and do not do something useful with the resulting memory, the compiler might optimize away the memory allocation. We’ve setup our Makefile to avoid this when using `malloc`.[3](#fn3)

Besides the memory allocation itself being optimized, accesses to that memory could also be optimized away. Probably the simplest way to avoid this is to make a pointer to `volatile char` (or `volatile int` or any other `volatile` type) and access values through that pointer.

### 2.2.2 accesses and resulting real allocations by malloc/calloc

It’s possible that `malloc` will access some memory to do bookkeeping This may cause additional pages to be allocated, increasing the resident set size even if you do no accesses — even though it is still the case that the memory is only allocated on demand. This may also mean that the virtual memory size may increase by sligthly more than you allocated to accomodate this bookkeeping information.

Also, sometimes `malloc` will keep memory it uses for bookkeeping near the allocation itself. In this case, it’s possible that if you values access near the beginning or end of the allocation, that your access will be part of the same page that `malloc` used and therefore caused to be allocated already.

2.3 Item 4 and Item 5
---------------------

### 2.3.1 find the location of the heap

1.  The memory layout we display indicates the range of the address that correspond to the heap (labeled `[heap]`)
    
    If you run the program with address randomization disabled, it should be the same every time.
    

### 2.3.2 mmap to allocate memory at a particular location

1.  On Linux, `mmap` can also be used for memory allocation, without involving any actual files. To do this one supplies the invalid file descriptor `-1` and specifies `MAP_ANONYMOUS` as one of the `mmap` flags. (The mapping is anonymous in the sense that there is no file name it corresponds to.) For example,
    
        char *ptr;
        ptr = mmap(NULL /* hint address */
                   8192 /* length */,
                   PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS,
                   -1, /* file descriptor (-1 for "none") */
                   0
            );
        if (ptr == MAP_FAILED) { handle_error(); }
    
    will make `ptr` point to a new allocation of 8192 bytes.
    
2.  By defaut `mmap` chooses a new location for the mapping.
    
    But you can use the MAP\_FIXED\_NOREPLACE or MAP\_FIXED flags to require the mapping be located in at a particular virtual address.
    
    For example:
    
        char *ptr;
        ptr = mmap((void*) 0x123456000,
                   8192,
                   PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS | MAP_FIXED_NOREPLACE,
                   -1,
                    0
            );
        if (ptr == MAP_FAILED) { handle_error(); }
    
    will allocate 8192 bytes of memory at virtual address 0x123456000 and set `ptr` to 0x123456000 if it succeeds. If something is already at that address or it fails for some other reason, `ptr` will be set to MAP\_FAILED.
    
    Unlike MAP\_FIXED\_NOREPLACE, using MAP\_FIXED will not fail if something is already allocated at the requested address; instead those address will be reassigned (probably breaking the program if anything important was there).
    
3.  Note that the location you specify must be at the beginning of a page, which on our systems is a multiple of 4096 (0x1000)
    

### 2.3.3 address randomization

By default most OSes, including Linux, randomizes the addresses of allocations.

This is typically done as a security feature. Although, ideally, it should not matter whether some attacking a program can guess the addresses of important code and data in the program, this information is helpful when there are other bugs in the program. For example, a common form of security bug might allow an attacker to execute code at an address they choose. If the addresses in the program are randomized, it is harder for an attacker to take advantage of that security bug.

The `setarch` command we mention allows running programs with this feature turned off.

### 2.3.4 why different increases

As we’ll learn in lecture and our readings later, the multi-level page table structure used on modern systems to store page table entries is more efficient when pages are closer to together.

There is a tree data structure where the bits of the virtual page number are used starting with the most significant bits to determine how to traverse the tree. This means that virtual page numbers that share more significant bits will share nodes of this tree. If all the virtual page numbers under a node of the tree are invalid, then that node does not need to be stored at all. Therefore, when more virtual page numbers that are valid have common significant bits, there will be fewer nodes and therefore fewer space devoted to storing the page table information.

### 2.3.5 mmap generally

1.  The `mmap` library function (documented in `man mmap`) was originally built to load a file’s data into a program’s memory. For example:
    
        int file_descriptor = open("foo.dat", O_RDWR);
        if (file_descriptor < 0) { handle_error(); }
        char *ptr;
        ptr = mmap(NULL /* "hint" address, NULL for "don't care what address chosen" */,
                   8192 /* length of mapping */,
                   PROT_READ | PROT_WRITE /* types of access allowed */,
                   MAP_SHARED /* "flags" controlling mapping */,
                   file_descriptor,
                   0 /* offset in file */
        );
        if (ptr == MAP_FAILED) { handle_error(); }
    
    will make `ptr` point to the first 8192 bytes (two pages on x86-64 Linux) of the file `foo.dat`. Since `MAP_SHARED` was selected, modifying what `ptr` points to will modify the file `foo.dat`. (The modifications will be shared with other users of the file; the alternate `MAP_PRIVATE` will give a private copy of the file’s data.)
    

3 Special regions in the memory layout
======================================

1.  When our program displays the memory layout, you may see some regions whose purpose is not obvious:
    
    *   dynamic allocation — this is an allocation of empty memory that is not on the heap. This can be used for global variables that are initially zero or by functions like malloc() when you request a large amount of memory. (It’s more efficient to make large allocations outside the heap, since then they can be freed without disrupting other things on the heap.)
        
    *   vvar, vdso — To speed up some operations that would otherwise be system calls, Linux implements a feature called [vDSO](https://man7.org/linux/man-pages/man7/vdso.7.html) (virtual dynamic shared object). The kernel loads a dynamically linked library into every program and this library has implementations of some operations which were previously system calls entirely in user-space.
        
        Some of these operations require access to data tracked by the OS, such as about how to translate the processor’s clock to a usable date and time. This information is shared with all processes through the vvar region.
        

* * *

1.  Originally I wrote access, and not write. You’ll see a change a the fault count with just reads. But to see a change in the resident set size, it needs to be a write. I think this is because Linux uses a clever trick of having a [common zero page](https://lwn.net/Articles/340370/) when the data is all zeroes.[↩︎](#fnref1)
    
2.  The `volatile` keyword is intended primarily for the purpose of supporting memory-mapped I/O, where reading and writing memory locations causes an input or output operation to occur instead of just storing or retrieving values. So, generally, assembly that the compiler generates will not omit reads and writes to volatile values even if they superficially appear to do nothing.
    
    An alternate, more direct strategy to avoid such optimizations would be to use assembly to do the memory accesses.[↩︎](#fnref2)
    
3.  By passing `-fno-builtin-malloc` to gcc or clang. Alternately you could prevent this from happening by using non-C-standard-library memory allocation functions like `mmap` or by accessing the resulting memory through `volatile` pointers (which one probably needs to do anyways as we describe above).[↩︎](#fnref3)
    

CS 3130 Fall 2024
-----------------

*   Charles Reiss
*   [\[email protected\]](/cdn-cgi/l/email-protection#147766717d676754627d66737d7a7d753a717061)

By Charles Reiss. Released under the [![Creative Commons License](https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png) CC-BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/).