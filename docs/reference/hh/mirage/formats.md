# Mirage File Format Specifications

## Basic Structure

Mirage file formats were designed to be serialized via "direct memory loading" - that is,
the concept of serializing data simply by reading an entire file's contents directly into a big buffer
in memory, and then fixing up the positions of its pointers (or "offsets"), at which point, the data can simply be used
directly with no further parsing or serialization required, just like a normal instance of a struct or class.

"Direct memory loading" has the major benefits of requiring little-to-no manual parsing work, and being
extremely fast. However, it also comes with a major limitation: the whole process can only work in the first
place if the data stored in the file **perfectly** matches, byte-for-byte, the layout that the program
expects the data to be stored in within memory.

Seeing as the Hedgehog Engine as a whole was seemingly designed heavily around the Xbox 360, this means
that all Mirage file formats were also designed specifically so they could be direct-memory-loaded on that platform.
Specifically, this means that all Mirage file formats:

- Store all of their data in **big-endian** byte order
- Utilize **32-bit pointers**

These pointers are represented within Mirage files as 32-bit unsigned integers that represent the distance,
in bytes, between the beginning of the data and the thing (or things) that the pointer is pointing to (aka "offsets").

!!! note
    Ironically, in practice, most games seem to ignore this intended "direct memory loading" approach entirely,
    and opt to just manually parse Mirage file formats instead. Hence, **the above applies even to 64-bit games
    and games on platforms which store utilize little-endian byte order**. Thanks, Sonic Team. 🙂

With that said, all Mirage files are structured in the following order:

1. Header
2. Format-Specific Data (as detailed [later in this document](#format-specific-data))
3. Offset Table
4. File Name (**Optional**, only present in like two files from Sonic Unleashed)

Let's start with the header.

### Standard Header

The "standard" Mirage header is as follows:

```c
struct MirageStandardHeader
{
    uint32_t fileSize;      /* Size of the entire file, including this header. */
    uint32_t version;       /* The version number for the format-specific data contained within the file. */
    uint32_t dataSize;      /* The size of the format-specific data contained within the file. */
    void* data;             /* Pointer to the format-specific data contained within the file. */
    uint32_t* offTable;     /* Pointer to the number of offsets in the offset table, followed immediately by the offset table itself. */
    char* fileName;         /* (Optional; can be NULL!!!) Pointer to a null-terminated string representing the name of the file, padded to a position divisible by 4. Probably used for debugging purposes? */
};
```

This is called the "Standard Header", because, actually, it's not the only possible header that can be used in Mirage file formats!
There's also the...

### Sample Chunk Header

Starting with Sonic Lost World, Mirage file formats can, optionally, begin with the new
"Sample Chunk Header" **instead of** the traditional "Standard Header".

!!! todo

### Offset Table

The "Offset Table" is an array of `#!c uint32_t` values representing the positions of each of the 32-bit pointers
(or "offsets") present within the format-specific data, relative to the beginning of said data.

The game is programmed to use the offset table after loading the file into memory to "fix" each of the
32-bit pointers contained within the file by transforming from "relative pointers" (pointers that are relative to the
beginning of the format-specific data) into "absolute pointers" (pointers that simply point directly to their targets).

Once this process is complete and all of the pointers in the file have been fixed, the pointers can simply be
used by the game like any other normal pointer used in their codebase.

Here's some psuedo-code that represents how the game does this:

```c
void FixPointers(void* data, const uint32_t* offTable, uint32_t offCount)
{
    uint32_t i;

    for (i = 0; i < offCount; i++)
    {
        /* Get a pointer to the current offset using the relative position in the offset table. */
        uint32_t* curOff = (uint32_t*)((uint8_t*)data + offTable[i]);

        /* "Fix" the current offset by adding the base address to it. */
        *curOff += (uint32_t)data;
    }
}
```

!!! note
    As mentioned earlier, actually, in practice, many Hedgehog Engine games don't actually bother doing
    all of this, and instead, simply manually jump to the positions pointed to by these pointers while
    manually parsing the data, completely defeating the point of the offset table in the first place.

## Format-Specific Data
### ResModel (.model files)

!!! todo
