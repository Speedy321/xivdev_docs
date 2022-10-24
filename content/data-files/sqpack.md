---
description: 'Everything SqPack: indexes, dat files'
---

# SqPack

SqPack\(s\) are formed from the following concepts, in which order 'matters':

1. [Repositories](sqpack.md#repositories)
2. [Categories](sqpack.md#categories)
3. [File Indexes](sqpack.md#reading-index-data)
4. Data Files

## Repositories

Repositories are essentially a collection of categories and there's not much to know here. As of writing, there's 4, one for the base game \(`ffxiv`\) and one for each expansion \(`ex1`, `ex2`, etc.\). Consider the following directory structure:

```text
sqpack/
    ex1/
    ex2/
    ex3/
    ffxiv/
```

Each folder inside `sqpack/` is it's own repository. As an aside, when the game tries to access files, it distinguishes which repository to find a file in by parsing the file path and pulling out 2 segments, the repository name and the category.

For example:

| Path                                                               | Repository | Category |
| :---                                                               | :---       | :---     |
| `bg/ex3/01_nvt_n4/twn/n4t1/bgparts/n4t1_a1_chr03.mdl`              | `ex3`      | `bg`     |
| `music/ex2/bgm_ex2_system_title.scd`                               | `ex2`      | `music`  |
| `chara/weapon/w0501/obj/body/b0018/vfx/texture/uv_cryst_128s.atex` | `ffxiv`    | `chara`  |

The first two paths explicitly define their repository in their file path, and it's always the second segment - if it's omitted it defaults to the `ffxiv` repository. Category is always the first segment and must be present for a path to resolve.

## Categories

Categories are just logical separations of game data. The following categories exist:

| ID   | Name          | Notes                                                                                                                                                                                                       |
| :--- | :---          | :---                                                                                                                                                                                                        |
| `0`  | `common`      | Contains basic data like fonts, vulgar words dictionary, shader input textures                                                                                                                              |
| `1`  | `bgcommon`    | Contains textures, models, environments and collision that are shared between territories                                                                                                                   |
| `2`  | `bg`          | Contains layouts, definitions, collision, models and textures for specific territories                                                                                                                      |
| `3`  | `cut`         | Contains cutscene animations and definitions                                                                                                                                                                |
| `4`  | `chara`       | Contains models, textures and definition files for all humans/demihumans/monsters                                                                                                                           |
| `5`  | `shader`      | Contains compiled shaders                                                                                                                                                                                   |
| `6`  | `ui`          | Contains UI layouts and textures                                                                                                                                                                            |
| `7`  | `sound`       | Contains sound effects                                                                                                                                                                                      |
| `8`  | `vfx`         | Contains textures and VFX definition files\(AVFX\)                                                                                                                                                          |
| `9`  | `ui_script`   | No index/dat in retail client, likely leftover from silverlight                                                                                                                                             |
| `A`  | `exd`         | Contains Excel [List](../game-data/file-formats/excel.md#excel-list-exl), [Header](../game-data/file-formats/excel.md#excel-header-exh) and [Data](../game-data/file-formats/excel.md#excel-data-exd) files |
| `B`  | `game_script` | Contains compiled Lua scripts (5.1) for quests, cutscenes and battles                                                                                                                                       |
| `C`  | `music`       | Contains BGM                                                                                                                                                                                                |
| `12` | `sqpack_test` | Category missing in retail client/no files.                                                                                                                                                                 |
| `13` | `debug`       | Category missing in retail client/no files.                                                                                                                                                                 |

Every game path will start with one of the above names and it defines which index to search to find a file.

## SqPack Files

Indexes and regular data files are effectively SqPack files and subsequently have a common header, `SqPackHeader`. It looks like the following:

{% tabs %}
{% tab title="C++" %}

```cpp
enum PlatformId : uint8_t
{
    Win32,
    PS3,
    PS4
};

// https://github.com/SapphireServer/Sapphire/blob/develop/deps/datReader/SqPack.cpp#L5
struct SqPackHeader
{
    char magic[0x8];
    PlatformId platformId;
    uint8_t padding0[3];
    uint32_t size;
    uint32_t version;
    uint32_t type;
};
```

{% endtab %}

{% tab title="C\#" %}

```csharp
public enum PlatformId : byte
{
    Win32,
    PS3,
    PS4
}

// https://github.com/NotAdam/Lumina/blob/master/Lumina/Data/Structs/SqPackHeader.cs
[StructLayout( LayoutKind.Sequential )]
public unsafe struct SqPackHeader
{
    public fixed byte magic[8];
    public PlatformId platformId;
    public fixed byte __unknown[3];
    public UInt32 size;
    public UInt32 version;
    public UInt32 type;
}
```

{% endtab %}
{% endtabs %}

Indexes \(and data\) starts at `size` so you want to seek to the value of `size` before you read anything out of the file.

## Reading Index Data

The index data is located directly after the header and depends on which variant of index file you load. The retail client ships with both variants of index files, which we'll mostly refer to as `index` and `index2` to make the difference obvious. Contrary to the retail client shipping with both index variants, benchmarks only ship with `index2` files. The reason is unknown.

Immediately following the `SqPackHeader` there's a `SqPackIndexHeader` \(which is only present in index files\):

{% tabs %}
{% tab title="C++" %}

```cpp
struct SqPackIndexHeader
{
    uint32_t size;
    uint32_t type;
    uint32_t indexDataOffset;
    uint32_t indexDataSize;
};
```

{% endtab %}

{% tab title="C\#" %}

```csharp
[StructLayout( LayoutKind.Sequential )]
public unsafe struct SqPackIndexHeader
{
    public UInt32 size;
    public UInt32 version;
    public UInt32 indexDataOffset;
    public UInt32 indexDataSize;
}
```

{% endtab %}
{% endtabs %}

The actual `SqPackIndexHeader` is `0x400`bytes large, but for the purposes of this, we're only interested in the first 16 bytes. From the `indexDataOffset` and `indexDataSize`, you can determine where to start reading from and how many index elements exist inside an index. `indexDataOffset` is an absolute offset to where the index data is located, and `indexDataSize` is the collective size of every `IndexHashTableEntry` that's in a file. This entry is slightly different in the case of `index2` files, so we'll generally cover the two with a focus on `index1` files for now.

### Reading Index

{% tabs %}
{% tab title="C++" %}

```cpp
struct IndexHashTableEntry
{
    uint64_t hash;
    uint32_t unknown : 1;
    uint32_t dataFileId : 3;
    uint32_t offset : 28;
    uint32_t _padding;
};
```

{% endtab %}

{% tab title="C\#" %}

```csharp
[StructLayout( LayoutKind.Sequential )]
public struct IndexHashTableEntry
{
    public UInt64 hash;
    public UInt32 data;
    private UInt32 _padding;

    public byte DataFileId => (byte) ( ( data & 0b1110 ) >> 1 );

    public uint Offset => (uint) ( data & ~0xF ) * 0x08;
}
```

{% endtab %}
{% endtabs %}

There's a couple notable differences between the C++ and C\# version, so we'll just explain the C++ version and the latter will make sense too.

The hash is a u64 that contains two u32s: the lower bits are the filename CRC32, the higher bits are the folder CRC32.

Generally speaking, calculating a `hash` works like this:

1. Convert the path to lowercase
2. Find the last instance of `/` and split the string with the last `/` existing in the first group. The filename needs to have no directory separators
3. Calculate the CRC32 of both path segments
4. Join both CRC32s into a u64, eg. `directoryHash << 32 | filenameHash`

The `dataFileId` is to identify which file \(on disk\) contains the file. Larger categories are split across multiple files \(each is capped at 2,000,000,000 bytes, or 2 GB\), so this is used to distinguish between `020000.win32.dat0` and `020000.win32.dat1` for example, where `dataFileId` would be `0` and `1` respectively for files located in either dat.

The `offset` is the absolute number of 8 byte aligned segments that the file is located at within a specific dat file. In a given dat file, a file is located at `offset * 0x8` which gives you the absolute offset to start reading a file from.

### Reading Index2

The main difference between `index` and `index2` is that the entire path is encoded into one CRC32 hash and does not split the path by folder and filename. Outside of that, everything is identical to `index`.

{% tabs %}
{% tab title="C++" %}

```cpp
struct Index2HashTableEntry
{
    uint32_t hash;
    uint32_t unknown : 1;
    uint32_t dataFileId : 3;
    uint32_t offset : 28;
};
```

{% endtab %}

{% tab title="C\#" %}

```csharp
[StructLayout( LayoutKind.Sequential )]
public struct Index2HashTableEntry
{
    public UInt32 hash;
    public UInt32 data;

    public byte DataFileId => (byte) ( ( data & 0b1110 ) >> 1 );

    public uint Offset => (uint) ( data & ~0xF ) * 0x08;
}
```

{% endtab %}
{% endtabs %}
