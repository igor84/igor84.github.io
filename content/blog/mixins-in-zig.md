---
title: "Mixins in Zig"
date: 2022-06-15T21:33:58+02:00
draft: false
hidden: true
tags: ["zig", "programming", "mixins"]
---

## What are mixins?

Mixins are a way to mix in some common functionality into multiple structs. For example if you have
a `File` and `TcpSocket` and they have their own different implementations of `read(buffer: []u8)`
method and you want to add convenience methods like `readInt()`, `readStruct()` and similar that
just call the `read()` method and format the result, you would usually have to write those methods
in one struct and then copy them to the other. Now if you find bugs in some of them you have to
remember to fix it in two places, or if you experiment with improving them you then need to remember
to apply the same improvements in other places, etc. Instead you can write these methods once in a
separate struct and then just mix them in, or in other words make them a part of other structs.

Currently the Zig standard library is not using this approach so there is another solution for this
but it has its own problems and I plan to analyze and propose alternatives to it in a future post.

Note that the example I gave only mixes in additional behavior while, in general, mixins might also
allow mixing in state, or in other words additional fields. In Zig you can't mix in additional
fields only functions and consts, but mixin code does have access to its modules global variables.
The cases where you actually need to mix in state are very rare and I couldn't come up with a single
non contrived example to show here.

## How are mixins done in Zig

Currently Zig has a `usingnamespace` keyword that will make all the consts and methods of the given
struct available in the current namespace, which in Zig is always another struct. Note that there is
a [proposal](https://github.com/ziglang/zig/issues/9861) to actually change it to `mixin` or
something like it so it might be different when you read this.

Let's see how would the example I gave above actually be implemented. First we will define the
methods we want to mix in.

```zig {hl_lines=[5]}
fn ReaderMethods(comptime Self: type) type {
    return struct{
        pub fn readInt(self: *Self) !i8 {
            var intBuffer: [4]u8 = undefined;
            _ = try self.read(intBuffer[0..]);
            return std.mem.bytesToValue(i32, intBuffer[0..]);
        }
        
        pub fn readStruct(comptime StructType: type) {
            // Only extern and packed structs have defined in-memory layout.
            comptime assert(@typeInfo(T).Struct.layout != .Auto);
            var res: [1]T = undefined;
            try self.read(mem.sliceAsBytes(res[0..]));
            return res[0];
        }
    }
}
```

So the first thing is that struct that you want to mix into other types has to be generic if you
want it to provide additional methods to target struct. The reason is that the first argument to
struct methods must be of the type of that struct and the only way for mixin code to know the type
of target struct is to pass it as a parameter of the generic function that generates it. The word
`comptime` before the parameter in Zig means that the value passed for that parameter must be known
at compile time. You can read more about it
[here](https://ziglang.org/documentation/master/#comptime) and
[here](https://ziglearn.org/chapter-1/#comptime).

You might also notice that the mixin function accepts any kind of type as its argument but later
code then expects that that type has a `read()` method on a highlighted fifth line. If a user passes
something that doesn't have a read method with that specific signature they will get a compile
error. Something like this:

```
error: no member named 'read' in struct 'TargetStruct'
```

The line on which the error is reported is the one in above mixin struct. In some situations this
might be completely fine but in others it might be better to provide a better error like this:

```zig
fn ReaderMethods(comptime Self: type, comptime ReadError: type) type {
    if (!@hasDecl(Self, "read")) {
        @compileError("Expected a read(*" ++ @typeName(Self) ++ ", []const u8) " ++
            @typeName(ReadError) ++ "!usize method in " ++ @typeName(Self));
    }
    if (@TypeOf(Self.read) != fn(*Self, []const u8) ReadError!usize) {
        @compileError("Expected a read(*" ++ @typeName(Self) ++ ", []const u8) " ++
            @typeName(ReadError) ++ "!usize method in " ++ @typeName(Self) ++
            " but got " ++ @typeName(@TypeOf(Self.read)));
    }
    return struct{
        …
    };
}
```

This will report the error on one of the `@compileError` lines but it will also show the line where
it was mixed in.

In order to mix the above behavior into a File struct you would just do this:

```zig
pub const File = struct {
    ...
    
    const Self = @This();
    
    pub fn read(self: *Self, buffer: []const u8) ReadError!usize {
        ...
    }
    
    pub usingnamespace ReaderMethods(Self, ReadError);
}
```

That one line does the job. You would add the same to `TcpSocket` and get the job done.

## Example with color types

Different image and graphic libraries support different formats of representing pixels. Most common
way is to store it as three values for red, green and blue channels. Still there could be a
different number of bits for each channel, there could additionally be an alpha channel and the
order of channels in memory could be different. How can we easily represent all those variations
with structs and also provide them all with some common conversion methods like `fromU32Rgba()` and
`toU32Rgba()`?

Using the mixins we can easily provide these methods:

```zig
fn RgbColor(comptime T: type) type {
    return packed struct {
        r: T,
        g: T,
        b: T,

        pub usingnamespace RgbMethods(@This());
    };
}

fn RgbaColor(comptime T: type) type {
    return packed struct {
        r: T,
        g: T,
        b: T,
        a: T = math.maxInt(T),

        pub usingnamespace RgbMethods(@This());
    };
}

fn BgrColor(comptime T: type) type {
    return packed struct {
        b: T,
        g: T,
        r: T,

        pub usingnamespace RgbMethods(@This());
    };
}

fn BgraColor(comptime T: type) type {
    return packed struct {
        b: T,
        g: T,
        r: T,
        a: T = math.maxInt(T),

        pub usingnamespace RgbMethods(@This());
    };
}

fn RgbMethods(comptime Self: type) type {
    return struct {
        const RedT = std.meta.fieldInfo(Self, .r).field_type;
        const GreenT = std.meta.fieldInfo(Self, .g).field_type;
        const BlueT = std.meta.fieldInfo(Self, .b).field_type;
        const AlphaT = RedT; // We assume Alpha type is same as Red type

        pub fn fromU32Rgba(value: u32) Self {
            var res = Self{
                .r = scaleToIntColor(RedT, @truncate(u8, value >> 24)),
                .g = scaleToIntColor(GreenT, @truncate(u8, value >> 16)),
                .b = scaleToIntColor(BlueT, @truncate(u8, value >> 8)),
            };
            if (@hasField(Self, "a")) res.a = scaleToIntColor(AlphaT, @truncate(u8, value));
            return res;
        }

        pub fn toU32Rgba(self: Self) u32 {
            return @as(u32, scaleToIntColor(u8, self.r)) << 24 |
                @as(u32, scaleToIntColor(u8, self.g)) << 16 |
                @as(u32, scaleToIntColor(u8, self.b)) << 8 |
                if (@hasField(Self, "a")) scaleToIntColor(u8, self.a) else 0xff;
        }
    };
}
```

In this example you can also see how we used `if (@hasField(Self, "a"))` in the mixin struct in
order to support alpha channel only if it exists in the target struct.

The `scaleToIntColor()` function makes sure that the final value is scaled to the target number of
bits even if the original value uses a different number of bits. For example, if u5 is used minimum
value will be 0 and maximum will be 31 but if we need to convert that to u8, 0 needs to stay 0 but
32 needs to become 255, while 16 will need to become 132, for example. This is how it looks like:

```zig
pub inline fn scaleToIntColor(comptime T: type, value: anytype) T {
    const ValueT = @TypeOf(value);

    const cur_value_bits = @bitSizeOf(ValueT);
    const new_value_bits = @bitSizeOf(T);
    if (cur_value_bits > new_value_bits) {
        return @truncate(T, value >> (cur_value_bits - new_value_bits));
    } else if (cur_value_bits < new_value_bits) {
        const cur_value_max = math.maxInt(ValueT);
        const new_value_max = math.maxInt(T);
        return @truncate(T, (@as(u32, value) * new_value_max + cur_value_max / 2) / cur_value_max);
    } else return @as(T, value);
}
```

So if we need to convert from bigger to smaller type we just shift the bits we don't need out using
right shift and then truncate to the smaller type. But if we need to convert from smaller to bigger
type we do some math in order to properly scale the value from a smaller range to a bigger range.

## Conditional mixin

Since Zig supports expressions that return types there is a way to use mixins to only define methods
in certain cases. For example lets say that we want `RgbMethods` above to also provide
`toPremultipliedAlpha()` method but only if target type has an alpha channel. We can do it like
this:

```zig
fn RgbMethods(comptime Self: type) type {
    return struct {
        ...
        
        pub usingnamespace if (@hasField(Self, "a")) struct {
            pub fn toPremultipliedAlpha(self: Self) Self {
                const max = math.maxInt(T);
                return Self{
                    .r = @truncate(RedT, (@as(u32, self.r) * self.a + max / 2) / max),
                    .g = @truncate(GreenT, (@as(u32, self.g) * self.a + max / 2) / max),
                    .b = @truncate(BlueT, (@as(u32, self.b) * self.a + max / 2) / max),
                    .a = self.a,
                };
            }
        } else struct {}
    };
}
```

Here we used inline mixin within RgbMethods mixin that we only mix in if target type has `a` field
:).

## Conclusion

This programming pattern is very powerful and can solve some tricky problems in a very elegant way.

One downside to these kind of solutions is that they generate a lot of code when compiled, because
now the compiler is copying that code for you. In some cases the optimizer might be able to
consolidate it if you are making an optimized build but there is no way to know upfront if that will
work. On the other hand that amount of compiled code will most likely be insignificant in any kind
of useful program so there are very limited cases where that size might be a factor.

Also note that Zig compiler will only generate code for definitions that are actually used by the
program and not for every possible definition which also reduces this problem a lot.

What do you think about this pattern? Do you see any other downsides to using it? If you interested
in discussing it join us on [Ziggit](https://forum.ziggit.dev/t/mixins-in-zig/208/) forum.