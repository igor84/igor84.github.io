---
title: "Png Reader in Zig"
date: 2022-05-15T21:00:00+02:00
draft: true
tags: ["zig", "png"]
hidden: true
---

Although there is already a solid [zigimg](https://github.com/zigimg/zigimg/) image loading library
it didn't quite tick all the checkboxes for me. It doesn't provide the flexibility that I am aiming
for and it is using far more allocations than is necessary. Also since I already wrote a png reader
in [DLang](https://dlang.org) it felt like a good project for learning zig to try to reimplement
that and try to improve on my original design.

In order to understand my design we first need to know a bit about the `png` format. Png file is a
sequence of chunks of different types. Each chunk has a 4 byte id, represented as 4 ascii letters.
Some examples are:

IHDR for a header chunk that contains things like width, height and pixel format PLTE for a palette
chunk in case pixels are stored as indices into that palette tRNS for a chunk that says what color
should be considered as transparent

The [specification](http://www.libpng.org/pub/png/spec/iso/index-object.html) says that if the first
letter of the id is upper case it is a critical chunk that the reader must know how to parse
otherwise the chunk is optional and image can be displayed without parsing it but it might not
displayed completely as intended.

So my idea was to create a reader that can parse critical chunks and allow users to register parsers
for any number of optional chunks. It would also come with some chunk parsers implemented but in
such a way that they are not even compiled in if they are not used. I didn’t quite manage to
accomplish that in the first version I wrote in DLang but I am pretty happy with what I managed to
write so far in Zig.

This is the API I came up with so far:

```zig
const file = try cwd.openFile("image.png", .{ .mode = .read_only });
var reader = pngreader.fromFile(file);
var header = try reader.loadHeader();
var pixelData = reader.loadWithHeader(&header, allocator, options);
```

In case you already have the whole file loaded into memory you can also use
`pngreader.fromMemory(u8buffer);`. Also if you don’t want to handle the header separately but just
load the image you can just call `reader.load(allocator, options)`.

The key to the design is, of course, the options argument. Its type is defined like this:

```zig
pub const ReaderOptions = struct {
    temp_allocator: Allocator,
    processors: []ReaderProcessor = &[_]ReaderProcessor{},
};
```

It provides a way for you to specify a separate allocator that will be used for temporary
allocations during loading and a slice of processors for different chunk types. Each
`ReaderProcessor` has an id which says what chunk type it is dedicated to and it provides
`processChunk` and optional `processPalette` and `processDataRow` functions. It is actually an
interface to specific implementation just like `Allocator` struct represents a common interface for
different implementations of allocators. You can register a processor for critical or optional chunk
but in case of optional chunks it will be a responsibility of its `processChunk` function to read in
or seek over the bytes of its chunk from underlining stream reader while for critical chunks it must
not do that but just use the passed in data. The other two methods allow each processor to affect
the palette or pixel data as they are loaded using the info they parsed.

Currently the reader comes with two processors implemented: `TrnsProcessor` and `PlteProcessor`.

`TrnsProcessor` will parse the tRNS chunk if it exists and extract what colors should be considered
transparent. It will then add an alpha channel to the image format and place that info there. So if
the image is in Grayscale format it will become GrayscaleAlpha and if it is in RGB format it will
become RGBA and in the alpha channel 0 will be written for a color that needs to be considered
transparent and 255 for others.

`PlteProcessor`, if the image is in Indexed format will change that to RGB format and will convert
pixel data so that instead of palette indices it contains the actual RGB values from the palette
directly. Note that if you also use `TrnsProcessor` and the tRNS chunk also exists the final format
will be RGBA since it will also add an alpha channel.

So if you just pass a default for the options argument:

```zig
var pixelData = reader.loadWithHeader(&header, allocator, options);
```

no processors will be used or compiled in and a static buffer on stack will be used for temporary
allocations. If you want to use default options that include the above two processors you can just
do this:

```zig
var pixelData = reader.loadWithHeader(&header, allocator, def_options.get());
```

It will also use a stack buffer for temporary allocations but it will use both of the above
processors.
