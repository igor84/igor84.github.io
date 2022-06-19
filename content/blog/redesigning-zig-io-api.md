---
title: "Redesigning Zig IO Api"
date: 2022-06-19T10:53:03+02:00
draft: false
hidden: true
tags: ["zig", "programming", "memory"]
---

Input-output being one of the most fundamental systems in any programming language was probably one of the first that was designed in Zig's standard library. As Zig grew and gained additional features it had a few redesigns but it is still not without issues so here I want to analyze how it is currently implemented and if we can now do better.

{{% notice style="note" %}}

There are additional language improvement proposals aiming to improve this exact area like [this](https://github.com/ziglang/zig/issues/1268 "comptime interfaces") and [this](https://github.com/ziglang/zig/issues/10184 "std.interface - better polymorphism support in standard library") but here I will only present what can be done with Zig today.

{{% /notice %}}

## Current Zig IO API

We will start at the lowest level with `std.fs.File` representing a file on the file system and `std.net.Stream` representing a network stream. Both of these have the following methods:

```zig
fn read(self, buffer: []u8) Error!u64
fn write(self, buffer: []const u8) Error!u64;
```
They also have a `close()` method and methods to fetch their `Reader` and `Writer` objects which we will explain later. `File` also has a lot of other methods that only make sense for a File but not for a network stream. One set of such methods, that are of interest, are seeking methods and a method to fetch a `SeekableStream` from a File. They are:

```zig
fn getEndPos(self: File) GetSeekPosError!u64;
fn getPos(self: File) GetSeekPosError!u64;
fn seekBy(self: File, offset: i64) SeekError!void;
fn seekTo(self: File, offset: i64) SeekError!void;
```

`std.io.SeekableStream` is a generic struct that contains a pointer to the original stream and those seekable methods that just call those same methods on that original stream. That way it acts as a sort of interface. `File`, of course, defines its own instance of this type so that the pointer in that instance is of type `*File` and same goes for `std.net.Stream`.

### Reader and Writer

`std.io.Reader` and `std.io.Writer` are also generic structs that contain a pointer to original stream but besides wrapping the `read()` and `write()` methods respectively they also provide additional methods, or, in other words, additional behavior. Some examples for Reader are:

```zig
fn readByte(self: @This()) !u8;
fn readInt(self: @This(), comptime T: type, endian: anytype) anytype;
fn readStruct(self: @This(), comptime T: type) anytype;
fn readUntilDelimiter(self: @This(), buf: []u8, delimiter: u8) ![]u8;
```

And some examples for Writer are:

```zig
fn print(self: @This(), comptime format: []const u8, args: anytype) anytype;
fn writeByte(self: @This(), byte: u8) !void;
fn writeByteNTimes(self: @This(), byte: u8, n: u64) !void
```

So the `Reader` and `Writer` wrap an existing method and provide additional ones while `SeekableStream` only wraps existing methods.

### StreamSource

When you write some kind of loaders or parsers you often want to support two ways of doing it. One way lets the user just specify the file from which to parse the data and another one allows the user to load the data themselves and then provide the buffer from which to parse.

Reading a buffer as a stream is provided by a `std.io.FixedBufferStream` which provides `read()`, `write()` and seek methods and methods to fetch the `Reader`, `Writer` and `SeekableStream` over that buffer.

That way if some loader function accepts any reader you can pass it either `File.reader()` or `FixefBufferStream.reader()` and it could work with both. Since those two readers are different type instances of a generic type that function would need to accept `anytype` and thus be generic itself. If it needs to store a reference to that reader then the entire type containing that function would need to be generic. That can lead to a lot of generated code.

`std.io.StreamSource` exists to solve that issue for the most common case I explained above. It is a union that can wrap either a buffer or a file and then provide one streaming API for both in the form of already described `read()`, `write()` and seek methods and methods to fetch the `Reader`, `Writer` and `SeekableStream`.

With it you can now write non generic loader functions that either accept the whole `StreamSource` or just `StreamSource.Reader` and they will support reading from either a file or memory buffer, but it will not, for example, support reading from `std.net.Stream` in any way.

## Analysis

`File` and `std.net.Stream` wrap the functionality offered by the operating systems with a simple and straightforward API so I don't see some room for options there. Maybe some of the methods that currently only exist on `File` could make sense for `std.net.Stream` as well, like `readAll` and `writeAll`, but that is it.

Now `Reader`, `Writer` and `SeekableStream` seem a bit odd. They look like interfaces but they are generic types, meaning each underlying stream that wants to provide them needs to define  its own specific type of those generics. That in turn means that any methods that want to receive any `Reader` for example, would have to actually receive a parameter of type `anytype` and then just assume or check that that type has all the `Reader` methods it needs. Same goes for the other two interfaces.

Having `SeekableStream` as a separate struct doesn't make sense from a usage point of view. You never just need a seeking functionality. You need it in combination with reading or writing. If some method needs a seekable reader it needs to receive `anytype` `Reader` and `anytype` `SeekableStream`. Two things that actually refer to the same underlying stream.

Currently `std.io.BufferedReader` is implemented in a way where it can wrap any other Reader and provide buffering additionally but if you then try to use it with a `SeekableStream` from the original stream it will not work. `BufferedReader` itself doesn't provide a way to do the seeking.

The only reason I see that it is now separated is the fact that some streams support seeking, like `File`, while some like `std.net.Stream` don't and there wasn't an easy way to sometimes create a `Reader` with and sometimes without seeking methods. Same goes for the `Writer`.

Another oddity of `SeekableStream` is that it doesn't add any additional behavior. As far as I see there is really no need for it at all in this form. If any method that needs a `SeekableStream` actually needs to receive `anytype` and then see if it has seek methods we can always just provide the original stream as a parameter since it will already have those methods. Currently they could be called differently in the original stream since they are actually passed as `comptime` parameters to generic `SeekableStream` struct but I saw no example where that was actually needed.

Another important problem of creating these `Reader`, `Writer` and `SeekableStream` abstractions over different types of streams is that those streams often return different error sets from their `read()`, `write()` and seek methods. Sometimes they don't even return any error at all.

That is the main reason those abstractions need to remain generic. The only other option is to allow methods in abstractions to return `anyerror` and thus lose information about specific possible errors.

## Alternatives

### 1. Just make `SeekableStream` a part of `Reader` and `Writer`

We can easily solve this with the help of mixins. If you don't know what they are or how they work in Zig you can read my previous [post](/blog/mixins-in-zig/).

The solution would look something like this for the `Reader`:

```zig
pub fn ReaderMethods(
    comptime Self: type,
    comptime ReadError: type,
    comptime readFn: fn (context: Context, buffer: []u8) ReadError!usize,
) type {
    return struct {
        pub const Error = ReadError;

        pub fn read(self: Self, buffer: []u8) Error!usize {
            return readFn(self.context, buffer);
        }

        // The rest of the Reader methods
    };
}

pub fn SeekMethods(comptime Self: type) type {
    return struct {
        pub const SeekError = getReturnErrorType(@TypeOf(self.context.seekBy));
        pub const GetSeekPosError = getReturnErrorType(@TypeOf(self.context.getPos));

        pub fn seekBy(self: Self, amt: i64) SeekError!void {
            return self.context.seekBy(amt);
        }

        // The rest of seek methods
    };
}

pub fn Reader(
    comptime Context: type,
    comptime ReadError: type,
    comptime readFn: fn (context: Context, buffer: []u8) ReadError!usize,
) type {
    return struct {
        context: Context,

        const Self = @This();

        pub usingnamespace ReaderMethods(Self, ReadError, readFn);

        // One option is to support seeking here directly if needed:
        pub usingnamespace if (hasSeekMethods(Context)) SeekMethods(Self) else struct {};
    };
}

// The other option is to provide explicit choice for SeekableReader:
pub fn SeekableReader(
    comptime Context: type,
    comptime ReadError: type,
    comptime readFn: fn (context: Context, buffer: []u8) ReadError!usize,
) type {
    return struct {
        context: Context,

        const Self = @This();

        pub usingnamespace ReaderMethods(Self, ReadError, readFn);
        pub usingnamespace SeekMethods(Self);
    };
}
```

Note that I am not passing seek methods explicitly like `SeekableStream` currently does since, as I said, I didn't find an example where they are called differently and can't be used directly. If there are other reasons to do that then we would need to go with the second option and pass another six parameters that `SeekableStream` now receives.

If we don't pass those methods we should probably add checks that Context does contain seek methods and report a nice `@compileError` if it doesn't.

Currently `BufferedReader` is defined like this:

```zig
pub fn BufferedReader(comptime buffer_size: usize, comptime ReaderType: type) type;
```

Just like above we can make `BufferedSeekMethods` mixin and mix it in only if `ReaderType` has seek methods. That way if passed in `ReaderType` has seek methods so will its `BufferedReader`.  That is one problem solved.

Additionally this would make it easy to write the `StreamSource` so that it wraps `File.BufferedReader` instead of `File` directly.

Another thing we can do is add something like this to the `Reader`:

```zig
pub const readerInterfaceId = @typeName(Context) ++ ".Reader";
```

If some method wants to check if the `anytype` parameter passed to it is a `Reader` it can just  check if it `@hasDecl(ReaderType, "readerInterfaceId")` and the value of that field could maybe be used in some `@compileError` messages. `SeekMethods` mixin could additionally add another `seekableIntefaceId` so we can use the same method to check if it also has seek methods.

The other problems coming from these being generic types remain but they might be insignificant.

Migrating the standard library and existing code to this solution shouldn't be too hard. Mostly it would involve deleting `SeekableStream` and its usages and std lib isn't using it anyway.

### 2. Use VTable interface like Allocator

Allocator is an interface to any concrete allocator implementation but it isn't a generic type. It is a plain struct that contains an opaque pointer to the actual allocator and a vtable struct with pointers to internal wrapping methods that also receive an opaque pointer as first parameter. Some comptime magic is only used to generate those internal wrapping methods so they cast that first parameter to the type of each concrete allocator implementation before calling its own method.

Allocator struct then also provides additional common utility methods that use some of the basic three that must be provided by the implementation: alloc, free and resize. So this method still allows adding additional behavior.

Now all allocator implementations can only return one error from their `alloc()` method and that is `OutOfMemory`. That allows this single `Allocator` struct to have a wrapping method that also can return just this error and still be the common interface for any concrete implementation.

As already mentioned, different streams return different kinds of errors from their `read`, `write` and seek methods so the only way this pattern could work is if the common interfaces return `anyerror`. That also means that any library function that works with a `Reader` or `Writer` also needs to return `anyerror` thus losing the ability to document and help its user properly handle its specific errors.

Because of this issue with errors this solution is really not viable so there is no point in further analyzing it.

### 3. Make streams be Readers and Writers using mixins

Again if you don't know what mixins are or how they are done in Zig you can read about it [here](/blog/mixins-in-zig/).

In this approach there would not be a separate `Reader`, `Writer` and `SeekableStream`. There would just be `ReaderMethods` and `WriterMethods` mixins that provide that common additional methods that currently `Reader` and `Writer` provide. Something like this:

```zig
pub fn ReaderMethods(comptime Self: type) type {
    // Now the provided Self type needs to provide the read method
    if (!@hasDecl(Self, "read")) {
        @compileError("Expected a read(*" ++ @typeName(Self) ++
            ", []const u8) !usize method in " ++ @typeName(Self));
    }
    const ReadError = getReturnErrorType(@TypeOf(Self.read));
    if (@TypeOf(Self.read) != fn(*Self, []u8) ReadError!usize) {
        @compileError("Expected a read " ++ @typeName(fn(*Self, []u8) ReadError!usize) ++
            " method in " ++ @typeName(Self) ++ " but got " ++ @typeName(@TypeOf(Self.read)));
    }
    return struct {
        pub fn readAll(self: Self, buffer: []u8) Error!usize {
            var index: usize = 0;
            while (index != buffer.len) {
                const amt = try self.read(buffer[index..]);
                if (amt == 0) return index;
                index += amt;
            }
            return index;
        }

        // The rest of the Reader methods
    };
}
```

`File` and `std.net.Stream` would now mixin those methods into their own implementation and `File` would still have those additional seeking methods. Functions that need to receive 'some seekable reader' for example, would provide one argument of type `anytype` and just like now check if all the methods needed, both reading and seeking, are defined on the given type.

We could use the trick I mentioned in the first alternative, where `ReaderMethods` also define

```zig
pub const readerInterfaceId = @typeName(Self) ++ ".Reader";
```

so that checking if some `ReaderType: anytype` is a reader can be done like this

```zig
if (@hasDecl(ReaderType, "readerInterfaceId")) ...
```

For that purpose we could also define `SeekerMethods()` mixin that only checks if the given `Self` type has all the seek methods and only mixes in

```zig
pub const seekInterfaceId = @typeName(Self) ++ ".Seeker";
```

and doesn't add any new methods.

`BufferedReader` would provide its own `read()` method that wraps the original one with additional logic just like now and would also mixin `ReaderMethods`. If the passed in reader has seek methods it could also wrap those using something like this:

```zig
pub usingnamespace if (@hasDecl(ReaderType, "seekInterfaceId")) struct {
    // Seek methods go here
} else struct {};
```

In this approach there is even less indirection than in the current implementation and it is also easy to implement and use. Methods that receive them as arguments being generic as in the current solution would also generate similar amounts of code.

One issue with this approach is that the struct that wants to use these mixins must have a read method with this exact signature:

```zig
read(self: *Self, buffer: []u8) SomeError!usize
```

Same goes for the `write()` method. Both `File` and `std.net.Stream` do have that but currently in std lib we also have:

1. `std.fifo.LinearFifo` which has a `read` method whose return type is just `usize` and not an error union and so for the current `Reader` it defines a separate method called `readFn` that just calls `read` but returns an error union with empty error set like this `error{}!usize`.
2. `std.os.uefi.protocols.FileProtocol` that has a `read` method with a bit different signature and then it also defines a separate `readFn` method for the `Reader` that has the proper signature and marshals the call to the actual `read`.

Note that the reason they must have a `read()` method that returns error union is not the check we added in `ReaderMethods` mixin but the fact that code that uses "any reader" might always call `try reader.read(...)` and the `try` statement will not compile unless `read()` returns an error union.

I am not sure how acceptable it would be to change the main `read()` method in those structs to align with the `Reader` interface and then provide their current `read()` method under a different name.

Migrating existing code to this alternative also wouldn't be too hard and would mostly involve deleting code. We would need to delete all `reader()`, `writer()` and `seekableStream()` methods and probably just replace their calls with the object they were called on. For example:

```zig
ImageLoader.load(someStream.reader());

// would just become

ImageLoader.load(someStream);

// and in case Image Loader was receiving StreamSource it would remain the same:
ImageLoader.load(someStreamSource);
```

`StreamSource` in this alternative would still wrap `read()`, `write()` and seek methods just like now and also mixin the `ReadMethods`, `WriteMethods` and `SeekMethods`.

## Conclusion

Only the second alternative turned out to be unusable but the analysis still helped us better understand why.

The first alternative that just merges `SeekableStream` into `Reader` and `Writer` seems to me like a clear improvement with no downsides over the existing solution.

Personally I like the third alternative the most. It gives the most potential for clearing up both the stream implementations and their usage code. I am just not sure how acceptable it is to require that every stream implements the exact `read()` and `write()` methods that are required by the interfaces.

What do you think? Do you have some further arguments to provide over some solution? Or maybe you have an idea for some new solution? Join us on [Ziggit](https://forum.ziggit.dev/t/redesigning-zig-io-api/210) forum and share your opinion.
