---
title: "Basics of Allocating and Using Memory"
date: 2022-06-13T14:05:23+02:00
draft: false
hidden: true
tags: ["zig", "programming", "memory"]
---

Many of the top used programming languages today use garbage collection and with them we are always
taught how manual memory management is hard. I get the feeling that many new programmers don't even
try to understand it today.

I want to show here how manual memory management can be not only very easy but also very fun.

## Lesson 1: Let Operating System Do It

There is a big class of programs that work as command line utilities. They run, do their job and
exit. Most of them don't need a lot of memory during their entire operation. In programs like this
you can act as if you have a garbage collector even when you don't because when your program exits
the operating system will clean up all the memory it used.

So the best strategy in these kinds of programs is to just allocate memory wherever you need and not
think about it.

## Lesson 2: What Is Memory Anyway

As soon as you go to a bit more complicated programs you need a better understanding of what memory
is and how it can be used in your programs.

First thing you need to learn is the difference between stack and heap memory.

### Stack memory

When you compile and link your code you produce an executable and in it will also be stored how much
stack memory it needs. There are ways to specify that manually while linking but linkers have a
default value for it. It is usually 8MB but on Windows it might be just 1MB.

When your program starts, the OS reserves that much space and sets up a special register to point to
the start of that memory. Now whenever you define a local variable in your program it gets stored on
stack. Also when you call some function the return address and all or some of the parameters are
also stored there. For example if in some function you define a local variable as an array of 100
integers and if one integer takes 4 bytes you just allocated 400 bytes from the stack memory. So now
how is that memory freed? Well since every local variable and function parameter can't be used when
you return from that function all the memory for that function is automatically freed.

So whenever the program enters a new function and new local variables are declared we say the stack
grows and whenever we return from a function we say the stack shrinks and that register that points
to the first free location is incremented or decremented.

That makes this method of allocation and deallocation the simplest one.

If you spend all the stack memory that was reserved for your program and you try to make another
function call the program will fail with Stack Overflow error.

Important to note is that compilers usually compile the code in such a way that as soon as a
function is entered the space for all local variables is immediately reserved. Even if some variable
is only used in some conditional block, space for it will be reserved at the start of the function.
That is why Stack Overflow errors happen only on entering new functions.

### Heap memory

This is usually additional memory that you can request from the OS and this is the memory that you
actually need to manage and properly release when no longer needed. It is used whenever you need to
store and keep something, even when you return from some function where it is created or if it is
too big to fit on the stack.

When you get this memory what you actually have is a pointer to the first address within it and how
much after that address is available. Some languages such as Zig represent those two things as one
thing called a slice. You can read more about Zig slices
[here](https://ziglearn.org/chapter-1/#slices).

Anyway it is up to you now how you want to interpret that memory. Is it one integer or an array of
integers, or maybe a float, a character or some structure. In zig you can use one of these
```zig
std.mem.bytesAsValue(comptime T: type, bytes: anytype)
std.mem.bytesToValue(comptime T: type, bytes: anytype)
```
to interpret the given array of bytes as a given `T` type.

If you are coming from object oriented and garbage collected language you might know that whenever
you call `new SomeClass()` the memory for that object is allocated and that object is created but
you can not imagine how you could control where and how it takes that memory. What the language does
for you is actually allocate memory for the data of that class, casts the pointer it gets to the
type of that class, then calls your constructor function and returns you that casted pointer. As you
can see you can easily do so yourself with just a struct and some function that initializes the
fields of that struct like a constructor would. Of course, unlike GC langauges you might also need
to manually release that memory when you no longer need it.

Now calling the OS whenever you need a few bytes of memory would ruin your performance because
system calls are expensive. That is why languages usually come with some sort of Allocators. 

## Lesson 3: Allocators

Allocators reserve more memory from the system than you currently need so they actually manage a
pool of memory that you can then efficiently borrow from whenever you need it. C only comes with
`malloc` which is a general purpose allocator but zig comes with more options.

General purpose allocator needs to try and be efficient no matter how you use it. Do you allocate a
lot of small objects or big ones or a mix of both, do you free often or rare, in small chunks or in
bulk. They also have to handle memory fragmentation.

If you imagine memory as a row of bytes where allocation reserves some of those bytes and freeing it
makes those bytes available for future use you can also imagine how after some usage there might be
a lot of little boxes that are available for use in between boxes that are still in use. Now if you
want to allocate something bigger there might be enough total free bytes but none of the boxes might
be big enough individually. That means your memory is fragmented and a lot of it becomes unusable.

Because of all of that general purpose allocators are the most complicated and often less efficient
than specialized allocators tailored for each use.

One example is `ArenaAllocator`. It doesn't support freeing individual allocations at all. The idea
is that you use it within a part of your program which does some processing and needs to do a number
of allocations none of which will be needed once the processing is complete. That way that part of
the program can just allocate things and not think about freeing them and once the processing is
complete you can just call 'free everything' on the ArenaAllocator itself. It is initialized with
another allocator from which the actual allocating and freeing is done.

Yet another is `FixedBufferAllocator`. It is initialized with a limited array of bytes that you want
to be served as memory from it. If the process then tries to allocate more memory from it than that
array contains it will get an error. It is useful to make sure parts of the program don't
accidentally allocate more than they should or that they don't leak memory. It is also useful
because you can initialize it with a local array that is on the stack and pass that as an allocator
to some library. You can only do this if the library has documented how much maximum memory it will
need to do its job and if that amount can fit on the stack. This way the actual allocation it does
through that allocator will be as cheap as possible and plus after its job is done all the memory is
guaranteed to be freed.

You can also use `StackFallbackAllocator` with a certain size, which will under the hood use a
`FixedBufferAllocator` on stack while the allocations fit in the given size and then start using a
provided fallback allocator for the rest of allocations.

Some others are `PageAllocator` which always allocates whole pages from the OS, which are usually
4KiB in size and `LoggingAllocator` that can log each allocation that happens through it.

Now with just these you now have a lot of tools in your arsenal you can play with.

## Examples

### Compiler

Compilers usually work in multiple passes. Each pass goes through data from the previous pass and
generates new or modifies existing data. Input for the first pass is of course a code file which can
also be thought of as a long string that needs to be parsed.

As each pass works it will need to allocate some temporary memory for its operation. For that
purpose it is best to allocate one block big enough to fit estimated temporary allocations of each
pass, maybe through `PageAllocator` and then make one `FixedBufferAllocator` over that memory that
you will call the `reset()` on after each pass. That will simply make the whole buffer it uses ready
to be used from start for the next pass thus reusing that initial allocation for each pass. It will
also limit the temporary memory each pass can take so you can detect memory leaks earlier.

For the things that one pass generates for the next to use it might be best to use `ArenaAllocator`.
That way if there is a pass in the pipeline that generates completely new data from the previous
ones you can just call `deinit()` on that allocator to free all the memory allocated from previous
passes at once.

Either way the memory that you need all the way until the end of the compiling process probably
doesn't need to be manually freed at all since the compiler process will shutdown when it is
finished and the OS will clean up all its memory anyway.

### Image loader

If you are writing a lib that needs to parse some file formats like png or jpg and return parsed
pixels it is best to allow the users to pass in the allocator that should be used. The catch is that
besides allocating space for resulting data the lib might need to allocate some temporary memory for
processing. It could just use the same allocator and in that case the best practice would be to
first allocate the space for the result and then start allocating and freeing the temporary data.
That way you minimize the possibility for fragmenting memory.

If you can guarantee the temporary space needed no matter the image size then you can internally
make a FixedBufferAllocator that will also quickly catch if you by mistake miss that target. If the
amount is also small you can just define a local array on the stack to serve as a backing buffer for
that allocator. Otherwise you would probably allocate the buffer from the given allocator.

Third option could be that you allow the user to pass in an additional allocator for temporary
allocations. It is best if you at least make it optional if you don't expect users will need that
level of customization often. You can still internally combine the methods above with that
allocator.

## Conclusion

Hopefully with just these two examples you can see that memory management can often be made very
simple. Also solving the issue of memory can be an interesting pursuit of simplicity and
flexibility.

Also note that although the principles described here should be applicable to any system programming
language each of them will have its own quirks, especially C and C++. The zig is the first language
I know that was designed not to have a globally accessible allocator, for example, which forces you
to be deliberate about your allocations and this post was written with it in mind.

If you want to learn more and practice, I strongly suggest you try the Zig language. Try to solve
some of the issues you worked on in the past using this language and its conventions and see how
much fun you find along the way ;).