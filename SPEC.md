## The little filesystem technical specification

This is the technical specification of the little filesystem. This document
covers the technical details of how the littlefs is stored on disk for
introspection and tool development. This document assumes you are familiar
with the design of the littlefs, for more info on how littlefs works check
out [DESIGN.md](DESIGN.md).

```
   | | |     .---._____
  .-----.   |          |
--|o    |---| littlefs |
--|     |---|          |
  '-----'   '----------'
   | | |
```

## Some quick notes

- littlefs is a block-based filesystem. The disk is divided into an array of
  evenly sized blocks that are used as the logical unit of storage. Block
  pointers are stored in 32 bits.

- In addition to the logical block size (which usually matches the erase
  block size), littlefs also uses a program block size and read block size.
  These determine the alignment of block device operations, but aren't needed
  for portability.

- By default, any values in littlefs are stored in little-endian byte order.

- The littlefs uses the value of `0xffffffff` to represent a null
  block address.

## Directories / Metadata pairs

Metadata pairs form the backbone of the littlefs and provide a system for
distributed atomic updates. Even the superblock is stored in a metadata pair.

As their name suggests, a metadata pair is stored in two blocks, with one block
providing a backup during erase cycles in case power is lost. These two blocks
are not necessarily sequential and may be anywhere on disk, so a "pointer" to a
metadata pair is stored as two block pointers.

On top of this, each metadata block behaves as an appendable log, containing a
variable number of commits. Commits can be appended to the metadata log in
order to update the metadata without requiring an erase cycles. Note that
successive commits may supersede the metadata in previous commits. Only the
most recent metadata should be considered valid.

The high-level layout of a metadata block is fairly simple:

```
  .----+----+----+----+----+----+----+----.
.-|  revision count   |      entries      |  \
| +----+----+----+----+                   |  |
| |                                       |  |
| |                                       |  +-- 1st commit
| |                                       |  |
| |                   +----+----+----+----+  |
| |                   |        CRC        |  /
| +----+----+----+----+----+----+----+----+
| |                entries                |  \
| |                                       |  |
| |                                       |  +-- 2nd commit
| |    +----+----+----+----+----+----+----+  |
| |    |        CRC        |    padding   |  /
| +----+----+----+----+----+----+----+----+
| |                entries                |  \
| |                                       |  |
| |                                       |  +-- 3rd commit
| |         +----+----+----+----+----+----|  |
| |         |        CRC        |         |  /
| +----+----+----+----+----+----+         |
| |           unwritten storage           |  more commits
| |                                       |       |
| |                                       |       v
| |                                       |
| |                                       |
| '----+----+----+----+----+----+----+----'
'---------------------------------------'
```

Each metadata block contains a 32-bit revision count followed by a number of
commits. Each commit contains a variable number of metadata entries followed
by a 32-bit CRC.

Note also that entries aren't necessarily word-aligned. This allows us to
store metadata more compactly, however we can only write to addresses that are
aligned to our program block size. This means each commit may have padding for
alignment.

Metadata block fields:

- | revision count | 32-bits |
  |----------------|---------|

  Incremented every erase cycle. If both blocks contain valid commits, only
  the block with the most recent revision count should be used. Sequence
  comparison must be used to avoid issues with integer overflow.

- | CRC            | 32-bits |
  |----------------|---------|

  Detects corruption from power-loss or other write issues. Uses the
  standard CRC-32, which uses a polynomial of `0x04c11db7`, initialized
  with `0xffffffff`.

Entries themselves are stored as a 32-bit tag followed by a variable length
blob of data. But exactly how these tags are stored is a little bit tricky.

Metadata blocks support both forward and backward iteration. In order to do
this without duplicating the space for each tag, neighboring entries have their
tags XORed together, starting with `0xffffffff`.

```
 Forward iteration                        Backward iteration

.-------------------.  0xffffffff        .-------------------.
|  revision count   |      |             |  revision count   |
|-------------------|      v             |-------------------|
|      tag ~A       |---> xor -> tag A   |      tag ~A       |---> xor -> 0xffffffff
|-------------------|      |             |-------------------|      ^
|       data A      |      |             |       data A      |      |
|                   |      |             |                   |      |
|                   |      |             |                   |      |
|-------------------|      v             |-------------------|      |
|      tag AxB      |---> xor -> tag B   |      tag AxB      |---> xor -> tag A
|-------------------|      |             |-------------------|      ^
|       data B      |      |             |       data B      |      |
|                   |      |             |                   |      |
|                   |      |             |                   |      |
|-------------------|      v             |-------------------|      |
|      tag BxC      |---> xor -> tag C   |      tag BxC      |---> xor -> tag B
|-------------------|                    |-------------------|      ^
|       data C      |                    |       data C      |      |
|                   |                    |                   |    tag C
|                   |                    |                   |
|                   |                    |                   |
'-------------------'                    '-------------------'
```

One last thing to note before we get into the details around tag encoding. Each
tag contains a valid bit used to indicate if the tag and containing commit is
valid. This valid bit is the first bit found in the tag and the commit and can
be used to tell if we've attempted to write to the remaining space in the
block.

Here's a more complete example of metadata block containing 4 entries:

```
  .----+----+----+----+----+----+----+----.
.-|  revision count   |      tag ~A       |        \
| +----+----+----+----+----+----+----+----+        |
| |                 data A                |        |
| |                                       |        |
| +----+----+----+----+----+----+----+----+        |
| |      tag AxB      |       data B      | <--.   |
| +----+----+----+----+                   |    |   |
| |                                       |    |   +-- 1st commit
| |         +----+----+----+----+----+----+    |   |
| |         |      tag BxC      |         | <-.|   |
| +----+----+----+----+----+----+         |   ||   |
| |                 data C                |   ||   |
| |                                       |   ||   |
| +----+----+----+----+----+----+----+----+   ||   |
| |     tag CxCRC     |        CRC        |   ||   /
| +----+----+----+----+----+----+----+----+   ||    
| |     tag CRCxA'    |      data A'      |   ||   \
| +----+----+----+----+                   |   ||   |
| |                                       |   ||   |
| |              +----+----+----+----+----+   ||   +-- 2nd commit
| |              |     tag CRCxA'    |    |   ||   |
| +----+----+----+----+----+----+----+----+   ||   |
| | CRC          |        padding         |   ||   /
| +----+----+----+----+----+----+----+----+   ||    
| |     tag CRCxA''   |      data A''     | <---.  \
| +----+----+----+----+                   |   |||  |
| |                                       |   |||  |
| |         +----+----+----+----+----+----+   |||  |
| |         |     tag A''xD     |         | < |||  |
| +----+----+----+----+----+----+         |  ||||  +-- 3rd commit
| |                data D                 |  ||||  |
| |                             +----+----+  ||||  |
| |                             |   tag Dx|  ||||  |
| +----+----+----+----+----+----+----+----+  ||||  |
| |CRC      |        CRC        |         |  ||||  /
| +----+----+----+----+----+----+         |  ||||   
| |           unwritten storage           |  ||||  more commits
| |                                       |  ||||       |
| |                                       |  ||||       v              
| |                                       |  ||||   
| |                                       |  |||| 
| '----+----+----+----+----+----+----+----'  ||||
'---------------------------------------'    |||'- most recent A
                                             ||'-- most recent B
                                             |'--- most recent C
                                             '---- most recent D
```

## Metadata tags

So in littlefs, 32-bit tags describe every type of metadata. And this means
_every_ type of metadata, including file entries, directory fields, and
global state. Even the CRCs used to mark the end of commits get their own tag.

Because of this, the tag format contains some densely packed informtaion. Note
that there are multiple levels of types which break down into more info:

```
[----            32             ----]
[1|--  11   --|--  10  --|--  10  --]
 ^.     ^     .     ^          ^- length
 |.     |     .     '------------ id
 |.     '-----.------------------ type (type3)
 '.-----------.------------------ valid bit
  [-3-|-- 8 --]
    ^     ^- chunk
    '------- type (type1)
```


Before we go further, there's one VERY important thing to note. These tags are
NOT stored in little-endian. Tags stored in commits are actually stored in
big-endian (and is the only thing in littlefs stored in big-endian). This
little bit of craziness comes from the fact that the valid bit must be the
first bit in a commit, and when converted to little-endian, the valid bit finds
itself in byte 4. We could restructure the tag to store the valid bit lower,
but, because none of the fields are byte-aligned, this would be more
complicated than just storing the tag in big-endian.

Another thing to note is that both the tags `0x00000000` and `0xffffffff` are
invalid and can be used for null values.

Metadata tag fields:

- | valid bit | 1-bit   |
  |-----------|---------|

  Indicates if the tag is valid.

- | type3     | 11-bits | `0x000` = invalid |
  |-----------|---------|-------------------|

  Type of the tag. This field is broken down further into a 3-bit abstract
  type and an 8-bit chunk field. Note that the value `0x000` is invalid and
  not assigned a type.

- | type1     | 3-bits  |
  |-----------|---------|

  Abstract type of the tag. Groups the tags into 8 categories that facilitate
  bitmasked lookups.

- | chunk     | 8-bits  |
  |-----------|---------|

  Chunk field used for various purposes by the different abstract types.
  type1+chunk+id form a unique identifier for each tag in the metadata block.

- | id        | 10-bits | `0x3ff` = no id |
  |-----------|---------|-----------------|

  File id associated with the tag. Each file in a metadata block gets a unique
  id which is used to associate tags with that file. The special value `0x3ff`
  is used for any tags that are not associated with a file, such as directory
  and global metadata.

- | length    | 10-bits | `0x3ff` = delete |
  |-----------|---------|------------------|

  Length of the data in bytes. The special value `0x3ff` indicates that this
  tag has been deleted.

## Metadata types

<!--
Each metadata tag falls into one of 7 abstract types:

<!-- | Encoding | Name              | Description                       | -->
<!-- |----------|-------------------|-----------------------------------| -->
<!-- | `0x4xx`  | LFS_TYPE_SPLICE   | Creation/deletion of files        | -->
<!-- | `0x0xx`  | LFS_TYPE_NAME     | Name and type of file             | -->
<!-- | `0x2xx`  | LFS_TYPE_STRUCT   | File data structure               | -->
<!-- | `0x3xx`  | LFS_TYPE_USERATTR | User attributes                   | -->
<!-- | `0x6xx`  | LFS_TYPE_TAIL     | Tail field for this metadata pair | -->
<!-- | `0x7xx`  | LFS_TYPE_GSTATE   | Global state                      | -->
<!-- | `0x5xx`  | LFS_TYPE_CRC      | CRC for the current commit        | -->

-->

What follows is an exhaustive list of metadata in littlefs.

#### `0x401` LFS_TYPE_CREATE

Creates a new file with this id. Note that files in a metadata block
don't necessarily need a create tag. All a create does is move over any
files using this id. In this sense a create is similar to insertion into
an imaginary array of files.

The create and delete tags allow littlefs to keep files in a directory
ordered alphabetically by filename.

#### `0x4ff` LFS_TYPE_DELETE

Deletes the file with this id. An inverse to create, this tag moves over
any files neighboring this id similar to a deletion from an imaginary
array of files.

#### `0x0xx` LFS_TYPE_NAME

Associates the id with a file name and file type. 

The data contains the file name stored as an ASCII string (may be expanded to
UTF8 in the future).

The chunk field in this tag indicates an 8-bit file type which can be one of
the following.

Currently, the name tag must precede any other tags associated with the id and
can not be reassigned without deleting the file.

#### `0x001` LFS_TYPE_REG

Initializes the id + name as a regular file.

How each file is stored depends on its struct tag, which is described below.

#### `0x002` LFS_TYPE_DIR

Initializes the id + name as a directory.

Directories in littlefs are stored on disk as a linked-list of metadata pairs,
each pair containing any number of files in alphabetical order. A pointer to
the directory is stored in the struct tag, which is described below.

#### `0x0ff` LFS_TYPE_SUPERBLOCK

Initializes the id as a superblock entry.

The superblock entry is a special entry used to store format-time configuration
and identify the filesystem.

The name is a bit of a misnomer. While the superblock entry serves the same
purpose as a superblock found in other filesystems, in littlefs the superblock
does not get a dedicated block. Instead, the superblock entry is duplicated
across a linked-list of metadata pairs rooted on the blocks 0 and 1. The last
metadata pair doubles as the root directory of the filesystem.

```
.--------.  .--------.  .--------.  .--------.  .--------.
| super  |->| super  |->| super  |->| super  |->| file B |
| block  |  | block  |  | block  |  | block  |  | file C |
|        |  |        |  |        |  | file A |  | file D |
'--------'  '--------'  '--------'  '--------'  '--------'

\---------------+----------------/  \---------+---------/
          superblock pairs              root directory
```

The filesystem starts with only the root directory. The superblock metadata
pairs grow every time the root pair is compacted in order to prolong the
life of the device exponentially.

The contents of the superblock entry are stored in a name tag with the
LFS_TYPE_SUPERBLOCK type and an inline-struct tag. The name tag contains
the magic string "littlefs", while the inline-struct tag contains version
and configuration information.

Layout of the superblock name tag and inline-struct tag:

```
[--      32       --]
[v| 0ff | 000 | 008 ]
.-------------------.
|   magic string    |
|    "littlefs"     |
'-------------------'
[v| 201 | 000 | 018 ]
.-------------------.
|      version      |
|-------------------|
|    block size     |
|-------------------|
|    block count    |
|-------------------|
|     name max      |
|-------------------|
|     file max      |
|-------------------|
|     attr max      |
'-------------------'
```

Superblock fields:

- | magic string | 8-bytes |
  |--------------|---------|

  Magic string indicating the presence of littlefs on the device. Must be the
  string "littlefs".

- | version      | 32-bits |
  |--------------|---------|

  The version of littlefs at format time. The version is encoded in a 32-bit
  value with the upper 16-bits containing the major version, and the lower
  16-bits containing the minor version.

  This specification describes version 2.0 (`0x00020000`).


- | block size   | 32-bits |
  |--------------|---------|

  Size of the logical block size used by the filesystem in bytes.

- | block count  | 32-bits |
  |--------------|---------|

  Number of blocks in the filesystem.

- | name max     | 32-bits |
  |--------------|---------|

  Maximum size of file names in bytes.

- | file max     | 32-bits |
  |--------------|---------|

  Maximum size of files in bytes.

- | attr max     | 32-bits |
  |--------------|---------|

  Maximum size of file attributes in bytes.

The superblock must always be the first entry (id 0) in a metdata pair as well
as be the first entry written to the block. This means that the superblock
entry can be read from a device using offsets alone.

```
  .----+----+----+----+----+----+----+----. \
.-|  revision count   |     name tag      | |
| +----+----+----+----+----+----+----+----+ +- magic string
| | "l   i    t    t    l    e    f    s" | |
| +----+----+----+----+----+----+----+----+ / \
| | inline struct tag |      version      |   |
| +----+----+----+----+----+----+----+----+   |
| |    block size     |    block count    |   |
| +----+----+----+----+----+----+----+----+   +- config info
| |     name max      |     file max      |   |
| +----+----+----+----+----+----+----+----+ \ |
| |     attr max      |      CRC tag      | | |
| +----+----+----+----+----+----+----+----+ | /
| |        CRC        |      padding      | +- CRC and padding
| +----+----+----+----+                   | |
| |                                       | |
| '---------------------------------------' /
'---------------------------------------'  
```

#### `0x2xx` LFS_TYPE_STRUCT

Associates the id with an on-disk data structure.

The exact layout of the data depends on the data structure type stored in the
chunk field and can be one of the following.

Any type of struct supercedes all other structs associated with the id. For
example, appending a ctz-struct replaces an inline-struct on the same file.

#### `0x200` LFS_TYPE_DIRSTRUCT

#### `0x201` LFS_TYPE_INLINESTRUCT

#### `0x202` LFS_TYPE_CTZSTRUCT

#### `0x3xx` LFS_TYPE_USERATTR

Attaches a user attribute to an id. 

littlefs has a concept of "user attributes". These are small user-provided
attributes that can be used to store things like timestamps, hashes,
permissions, etc.

Each user attribute is uniquely identified by an 8-bit type which is stored in
the chunk field, and the user attribute itself can be found in the tag's data.

There are currently no standard user attributes and a portable littlefs
implementation should work with any user attributes missing.

#### `0x6xx` LFS_TYPE_TAIL

Provides data for any state associated with the metadata pair itself.

Currently, this is only used for is the metadata pair's tail pointer, which is
used in the filesystem for a linked-list containing all metadata blocks.

The type of the tail is stored in the chunk field. Currently any type
supercedes any other preceding tails in the metadata pair, but this may change
if additional metadata pair state is added.

#### `0x600` LFS_TYPE_SOFTTAIL



```
         .--------.  
         | dir A  |-.
         |softtail| |
.--------|        |-'
|        '--------'  
|        .-'    '-------------.
|       v                      v
|  .--------.  .--------.  .--------.
'->| dir B  |->| dir B  |->| dir C  |
   |        |  |softtail|  |        |
   |        |  |        |  |        |
   '--------'  '--------'  '--------'
```

#### `0x601` LFS_TYPE_HARDTAIL

```
         .--------.  
         | dir A  |-.
         |        | |
.--------|        |-'
|        '--------'  
|        .-'    '-------------.
|       v                      v
|  .--------.  .--------.  .--------.
'->| dir B  |->| dir B  |->| dir C  |
   |hardtail|  |        |  |        |
   |        |  |        |  |        |
   '--------'  '--------'  '--------'
```

#### `0x7xx` LFS_TYPE_GSTATE

Provides delta bits for global state entries.

littlefs has a concept of "global state". This is a small set of state that
can be updated by a commit to _any_ metadata pair in the filesystem.

The way this works is that the global state is stored as a set of deltas
distributed across the filesystem such that the global state can by found by
the xor-sum of these deltas.

```
.--------.  .--------.  .--------.  .--------.  .--------.
|        |->| gstate |->|        |->| gstate |->| gstate |
|        |  | 0x23   |  |        |  | 0xff   |  | 0xce   |
|        |  |        |  |        |  |        |  |        |
'--------'  '--------'  '--------'  '--------'  '--------'
                |                       |           |
                v                       v           v
      0x00 --> xor ------------------> xor ------> xor --> gstate 0x12
```

Note that storing globals this way is very expensive in terms of storage usage,
so any global state should be kept very small.

The size and format of each piece of global state depends on the type, which
is stored in the chunk field. Currently, the only global state is move state,
which is outlined below.

#### `0x7ff` LFS_TYPE_MOVESTATE



#### `0x5xx` LFS_TYPE_CRC

Last but not least, the CRC tag marks the end of a commit and provides a
checksum for any commits to the metadata block.

The first 32-bits of the data contain a CRC-32 with a polynomial of
`0x04c11db7` initialized to `0xffffffff`. This CRC provides a checksum for all
metadata since the previous CRC tag, including the CRC tag itself. For the
first commit, this includes the revision count for the metadata block.

However, the size of the data is not limited to 32-bits. The data field may
larger to pad the commit to the next program-aligned boundary.

In addition, the CRC tag's chunk field contains a set of flags which can
change the behaviour of commits. Currently the only flag in use is the lowest
bit, which determines the expected state of the valid bit for any following
tags. This is used to guarantee that unwritten storage in a metadata block
will be detected as invalid.

# TODO RM below me

















## Metadata tags

These tag+data pairs describe every type of metadata used in littlefs.











<table>
<tr>
<th>Name</th>
<th>Size</th>
<th>Description</th>
</tr>
<tr>
<td>Revision&nbsp;count</td>
<td>32&nbsp;bits</td>
<td>Incremented every erase cycle. If both blocks contain valid commits, only the block with the most recent revision count should be used. Sequence comparison must be used to avoid issues with integer overflow.</td>
</tr>
<tr>
<td>CRC</td>
<td>32&nbsp;bits</td>
<td>Detects corruption from power-loss or other write issues. Uses the standard CRC-32, which uses a polynomial of 0x04c11db7, initialized with 0xffffffff.</td>
</tr>
</table>




<table>
<tr>

| name           | size            | description                |
|----------------|-----------------|----------------------------|
| revision count | 32 bits         | Incremented every erase cycle. If both blocks contain valid commits, the block with the highest                           |
| entries        | variable length | 
| CRC            | 32 bits         |                            |
| 

<table>
<tr>
<th>Name</th>
<th>Size</th>
<th>Description</th>
</tr>
<tr>
<td>Revision count</td>
<td>32 bits</td>
<td>Incremented every erase cycle. If both blocks contain valid commits, only
the block with the most recent revision count should be used. Sequence
comparison must be used to avoid issues with integer overflow.</td>
</tr>
<tr>
<td>entries</td>
<td>variable length</td>
<td>Variable number of entries containing metadata related to the
file-system.</td>
</tr>
<tr>
<td>CRC</td>
<td>32 bits</td>
<td>Detects corruption from power-loss or other write issues. Uses the standard
CRC-32, which uses a polynomial of 0x04c11db7, initialized with
0xffffffff.</td>
</tr>
</table>







Each metadata block 

Each metadata block is an appendable log 



Metadata block structure:

| name           | size            | description                |
|----------------|-----------------|----------------------------|
| revision count | 32 bits         |                            |
| commits        | variable length |                            |

Metadata commit structure:

| name           | size            | description                |
|----------------|-----------------|----------------------------|
| entries        | variable length |                            |
| crc            | 32 bits         |                            |

Metadata entry structure:

| name           | size            | description                |
|----------------|-----------------|----------------------------|
| tag            | 32 bits         |                            |
| data           | variable length |                            |





| tags           | variable length | 
| crc            | 32 bits         |





| offset  | size                   | description                |
|---------|------------------------|----------------------------|
| 0x0     | 8 bits                 | entry type                 |
| 0x1     | 8 bits                 | entry length               |
| 0x2     | 8 bits                 | attribute length           |
| 0x3     | 8 bits                 | name length                |
| 0x4     | entry length bytes     | entry-specific data        |
| 0x4+e   | attribute length bytes | system-specific attributes |
| 0x4+e+a | name length bytes      | entry name                 |









```
.----+----+----+----+----+----+----+----.
|  revision count   |      tag ~A       |        \
+----+----+----+----+----+----+----+----+        |
|                 data A                |        |
|                                       |        |
+----+----+----+----+----+----+----+----+        |
|      tag AxB      |       data B      | <--\   |
+----+----+----+----+                   |    |   |
|                                       |    |   +-- 1st commit
|         +----+----+----+----+----+----+    |   |
|         |      tag BxC      |         | <-\|   |
+----+----+----+----+----+----+         |   ||   |
|                 data C                |   ||   |
|                                       |   ||   |
+----+----+----+----+----+----+----+----+   ||   |
|     tag CxCRC     |        CRC        |   ||   /
+----+----+----+----+----+----+----+----+   ||    
|     tag CRCxA'    |      data A'      |   ||   \
+----+----+----+----+                   |   ||   |
|                                       |   ||   |
|              +----+----+----+----+----+   ||   +-- 2nd commit
|              |     tag CRCxA'    |    |   ||   |
+----+----+----+----+----+----+----+----+   ||   |
| CRC          |        padding         |   ||   /
+----+----+----+----+----+----+----+----+   ||    
|     tag CRCxA''   |      data A''     | <---\  \
+----+----+----+----+                   |   |||  |
|                                       |   |||  |
|         +----+----+----+----+----+----+   |||  |
|         |     tag A''xD     |         | < |||  |
+----+----+----+----+----+----+         |  ||||  +-- 3rd commit
|                data D                 |  ||||  |
|                             +----+----+  ||||  |
|                             |   tag Dx|  ||||  |
+----+----+----+----+----+----+----+----+  ||||  |
|CRC      |        CRC        |         |  ||||  /
+----+----+----+----+----+----+         |  ||||   
|           unwritten storage           |  ||||  more commits
|                                       |  ||||       |
|                                       |  ||||       v              
|                                       |  ||||   
|                                       |  |||| 
'----+----+----+----+----+----+----+----'  ||||
                                           |||\- most recent A
                                           ||\-- most recent B
                                           |\--- most recent C
                                           \---- most recent D
```


```
[--     32     --|--     32     --|--          ??           --|
[ revision count |      tag       |           data            |      tag       |



```


```
.----+----+----+----+----+----+----+----. 
|  revision count   |        tags       |  \
+----+----+----+----+                   |  |
|                                       |  |
|                                       |  +-- 1st commit
|                                       |  |
|         +----+----+----+----+----+----+
|         |        CRC        | padding |  /
+----+----+----+----+----+----+----+----+
|        tags                           |  \
|                                       |  |
|                                       |  +-- 2nd commit
|                   +----+----+----+----+  |
|                   |        CRC        |  /
+----+----+----+----+----+----+----+----+
|        tags                           |  \
|                                       |  |
|                                       |  +-- 3rd commit
|                   +----+----+----+----|  |
|                   |        CRC        |  /
+----+----+----+----+----+----+----+----+
|                padding                |  more commits 
|                                       |       |
|                                       |       v
|                                       |
|                                       |
'----+----+----+----+----+----+----+----'
```



Here's the layout of metadata blocks on disk:

| offset | size          | description    |
|--------|---------------|----------------|
| 0x00   | 32 bits       | revision count |
| 0x04   | 32 bits       | dir size       |
| 0x08   | 64 bits       | tail pointer   |
| 0x10   | size-16 bytes | dir entries    |
| 0x00+s | 32 bits       | CRC            |

**Revision count** - Incremented every update, only the uncorrupted
metadata block with the most recent revision count contains the valid metadata.
Comparison between revision counts must use sequence comparison since the
revision counts may overflow.

**Dir size** - Size in bytes of the contents in the current metadata block,
including the metadata pair metadata. Additionally, the highest bit of the
dir size may be set to indicate that the directory's contents continue on the
next metadata pair pointed to by the tail pointer.

**Tail pointer** - Pointer to the next metadata pair in the filesystem.
A null pair-pointer (0xffffffff, 0xffffffff) indicates the end of the list.
If the highest bit in the dir size is set, this points to the next
metadata pair in the current directory, otherwise it points to an arbitrary
metadata pair. Starting with the superblock, the tail-pointers form a
linked-list containing all metadata pairs in the filesystem.

**CRC** - 32 bit CRC used to detect corruption from power-lost, from block
end-of-life, or just from noise on the storage bus. The CRC is appended to
the end of each metadata block. The littlefs uses the standard CRC-32, which
uses a polynomial of 0x04c11db7, initialized with 0xffffffff.

Here's an example of a simple directory stored on disk:
```
(32 bits) revision count = 10                    (0x0000000a)
(32 bits) dir size       = 154 bytes, end of dir (0x0000009a)
(64 bits) tail pointer   = 37, 36                (0x00000025, 0x00000024)
(32 bits) CRC            = 0xc86e3106

00000000: 0a 00 00 00 9a 00 00 00 25 00 00 00 24 00 00 00  ........%...$...
00000010: 22 08 00 03 05 00 00 00 04 00 00 00 74 65 61 22  "...........tea"
00000020: 08 00 06 07 00 00 00 06 00 00 00 63 6f 66 66 65  ...........coffe
00000030: 65 22 08 00 04 09 00 00 00 08 00 00 00 73 6f 64  e"...........sod
00000040: 61 22 08 00 05 1d 00 00 00 1c 00 00 00 6d 69 6c  a"...........mil
00000050: 6b 31 22 08 00 05 1f 00 00 00 1e 00 00 00 6d 69  k1"...........mi
00000060: 6c 6b 32 22 08 00 05 21 00 00 00 20 00 00 00 6d  lk2"...!... ...m
00000070: 69 6c 6b 33 22 08 00 05 23 00 00 00 22 00 00 00  ilk3"...#..."...
00000080: 6d 69 6c 6b 34 22 08 00 05 25 00 00 00 24 00 00  milk4"...%...$..
00000090: 00 6d 69 6c 6b 35 06 31 6e c8                    .milk5.1n.
```

A note about the tail pointer linked-list: Normally, this linked-list is
threaded through the entire filesystem. However, after power-loss this
linked-list may become out of sync with the rest of the filesystem.
- The linked-list may contain a directory that has actually been removed
- The linked-list may contain a metadata pair that has not been updated after
  a block in the pair has gone bad.

The threaded linked-list must be checked for these errors before it can be
used reliably. Fortunately, the threaded linked-list can simply be ignored
if littlefs is mounted read-only.

## Entries

Each metadata block contains a series of entries that follow a standard
layout. An entry contains the type of the entry, along with a section for
entry-specific data, attributes, and a name.

Here's the layout of entries on disk:

| offset  | size                   | description                |
|---------|------------------------|----------------------------|
| 0x0     | 8 bits                 | entry type                 |
| 0x1     | 8 bits                 | entry length               |
| 0x2     | 8 bits                 | attribute length           |
| 0x3     | 8 bits                 | name length                |
| 0x4     | entry length bytes     | entry-specific data        |
| 0x4+e   | attribute length bytes | system-specific attributes |
| 0x4+e+a | name length bytes      | entry name                 |

**Entry type** - Type of the entry, currently this is limited to the following:
- 0x11 - file entry
- 0x22 - directory entry
- 0x2e - superblock entry

Additionally, the type is broken into two 4 bit nibbles, with the upper nibble
specifying the type's data structure used when scanning the filesystem. The
lower nibble clarifies the type further when multiple entries share the same
data structure.

The highest bit is reserved for marking the entry as "moved". If an entry
is marked as "moved", the entry may also exist somewhere else in the
filesystem. If the entry exists elsewhere, this entry must be treated as
though it does not exist.

**Entry length** - Length in bytes of the entry-specific data. This does
not include the entry type size, attributes, or name. The full size in bytes
of the entry is 4 + entry length + attribute length + name length.

**Attribute length** - Length of system-specific attributes in bytes. Since
attributes are system specific, there is not much guarantee on the values in
this section, and systems are expected to work even when it is empty. See the
[attributes](#entry-attributes) section for more details.

**Name length** - Length of the entry name. Entry names are stored as UTF8,
although most systems will probably only support ASCII. Entry names can not
contain '/' and can not be '.' or '..' as these are a part of the syntax of
filesystem paths.

Here's an example of a simple entry stored on disk:
```
(8 bits)   entry type       = file     (0x11)
(8 bits)   entry length     = 8 bytes  (0x08)
(8 bits)   attribute length = 0 bytes  (0x00)
(8 bits)   name length      = 12 bytes (0x0c)
(8 bytes)  entry data       = 05 00 00 00 20 00 00 00
(12 bytes) entry name       = smallavacado

00000000: 11 08 00 0c 05 00 00 00 20 00 00 00 73 6d 61 6c  ........ ...smal
00000010: 6c 61 76 61 63 61 64 6f                          lavacado
```

## Superblock

The superblock is the anchor for the littlefs. The superblock is stored as
a metadata pair containing a single superblock entry. It is through the
superblock that littlefs can access the rest of the filesystem.

The superblock can always be found in blocks 0 and 1, however fetching the
superblock requires knowing the block size. The block size can be guessed by
searching the beginning of disk for the string "littlefs", although currently
the filesystems relies on the user providing the correct block size.

The superblock is the most valuable block in the filesystem. It is updated
very rarely, only during format or when the root directory must be moved. It
is encouraged to always write out both superblock pairs even though it is not
required.

Here's the layout of the superblock entry:

| offset | size                   | description                            |
|--------|------------------------|----------------------------------------|
| 0x00   | 8 bits                 | entry type (0x2e for superblock entry) |
| 0x01   | 8 bits                 | entry length (20 bytes)                |
| 0x02   | 8 bits                 | attribute length                       |
| 0x03   | 8 bits                 | name length (8 bytes)                  |
| 0x04   | 64 bits                | root directory                         |
| 0x0c   | 32 bits                | block size                             |
| 0x10   | 32 bits                | block count                            |
| 0x14   | 32 bits                | version                                |
| 0x18   | attribute length bytes | system-specific attributes             |
| 0x18+a | 8 bytes                | magic string ("littlefs")              |

**Root directory** - Pointer to the root directory's metadata pair.

**Block size** - Size of the logical block size used by the filesystem.

**Block count** - Number of blocks in the filesystem.

**Version** - The littlefs version encoded as a 32 bit value. The upper 16 bits
encodes the major version, which is incremented when a breaking-change is
introduced in the filesystem specification. The lower 16 bits encodes the
minor version, which is incremented when a backwards-compatible change is
introduced. Non-standard Attribute changes do not change the version. This
specification describes version 1.1 (0x00010001), which is the first version
of littlefs.

**Magic string** - The magic string "littlefs" takes the place of an entry
name.

Here's an example of a complete superblock:
```
(32 bits) revision count   = 3                    (0x00000003)
(32 bits) dir size         = 52 bytes, end of dir (0x00000034)
(64 bits) tail pointer     = 3, 2                 (0x00000003, 0x00000002)
(8 bits)  entry type       = superblock           (0x2e)
(8 bits)  entry length     = 20 bytes             (0x14)
(8 bits)  attribute length = 0 bytes              (0x00)
(8 bits)  name length      = 8 bytes              (0x08)
(64 bits) root directory   = 3, 2                 (0x00000003, 0x00000002)
(32 bits) block size       = 512 bytes            (0x00000200)
(32 bits) block count      = 1024 blocks          (0x00000400)
(32 bits) version          = 1.1                  (0x00010001)
(8 bytes) magic string     = littlefs
(32 bits) CRC              = 0xc50b74fa

00000000: 03 00 00 00 34 00 00 00 03 00 00 00 02 00 00 00  ....4...........
00000010: 2e 14 00 08 03 00 00 00 02 00 00 00 00 02 00 00  ................
00000020: 00 04 00 00 01 00 01 00 6c 69 74 74 6c 65 66 73  ........littlefs
00000030: fa 74 0b c5                                      .t..
```

## Directory entries

Directories are stored in entries with a pointer to the first metadata pair
in the directory. Keep in mind that a directory may be composed of multiple
metadata pairs connected by the tail pointer when the highest bit in the dir
size is set.

Here's the layout of a directory entry:

| offset | size                   | description                             |
|--------|------------------------|-----------------------------------------|
| 0x0    | 8 bits                 | entry type (0x22 for directory entries) |
| 0x1    | 8 bits                 | entry length (8 bytes)                  |
| 0x2    | 8 bits                 | attribute length                        |
| 0x3    | 8 bits                 | name length                             |
| 0x4    | 64 bits                | directory pointer                       |
| 0xc    | attribute length bytes | system-specific attributes              |
| 0xc+a  | name length bytes      | directory name                          |

**Directory pointer** - Pointer to the first metadata pair in the directory.

Here's an example of a directory entry:
```
(8 bits)  entry type        = directory (0x22)
(8 bits)  entry length      = 8 bytes   (0x08)
(8 bits)  attribute length  = 0 bytes   (0x00)
(8 bits)  name length       = 3 bytes   (0x03)
(64 bits) directory pointer = 5, 4      (0x00000005, 0x00000004)
(3 bytes) name              = tea

00000000: 22 08 00 03 05 00 00 00 04 00 00 00 74 65 61     "...........tea
```

## File entries

Files are stored in entries with a pointer to the head of the file and the
size of the file. This is enough information to determine the state of the
CTZ skip-list that is being referenced.

How files are actually stored on disk is a bit complicated. The full
explanation of CTZ skip-lists can be found in [DESIGN.md](DESIGN.md#ctz-skip-lists).

A terribly quick summary: For every nth block where n is divisible by 2^x,
the block contains a pointer to block n-2^x. These pointers are stored in
increasing order of x in each block of the file preceding the data in the
block.

The maximum number of pointers in a block is bounded by the maximum file size
divided by the block size. With 32 bits for file size, this results in a
minimum block size of 104 bytes.

Here's the layout of a file entry:

| offset | size                   | description                        |
|--------|------------------------|------------------------------------|
| 0x0    | 8 bits                 | entry type (0x11 for file entries) |
| 0x1    | 8 bits                 | entry length (8 bytes)             |
| 0x2    | 8 bits                 | attribute length                   |
| 0x3    | 8 bits                 | name length                        |
| 0x4    | 32 bits                | file head                          |
| 0x8    | 32 bits                | file size                          |
| 0xc    | attribute length bytes | system-specific attributes         |
| 0xc+a  | name length bytes      | directory name                     |

**File head** - Pointer to the block that is the head of the file's CTZ
skip-list.

**File size** - Size of file in bytes.

Here's an example of a file entry:
```
(8 bits)   entry type       = file     (0x11)
(8 bits)   entry length     = 8 bytes  (0x08)
(8 bits)   attribute length = 0 bytes  (0x00)
(8 bits)   name length      = 12 bytes (0x03)
(32 bits)  file head        = 543      (0x0000021f)
(32 bits)  file size        = 256 KB   (0x00040000)
(12 bytes) name             = largeavacado

00000000: 11 08 00 0c 1f 02 00 00 00 00 04 00 6c 61 72 67  ............larg
00000010: 65 61 76 61 63 61 64 6f                          eavacado
```

## Entry attributes

Each dir entry can have up to 256 bytes of system-specific attributes. Since
these attributes are system-specific, they may not be portable between
different systems. For this reason, all attributes must be optional. A minimal
littlefs driver must be able to get away with supporting no attributes at all.

For some level of portability, littlefs has a simple scheme for attributes.
Each attribute is prefixes with an 8-bit type that indicates what the attribute
is. The length of attributes may also be determined from this type. Attributes
in an entry should be sorted based on portability, since attribute parsing
will end when it hits the first attribute it does not understand.

Each system should choose a 4-bit value to prefix all attribute types with to
avoid conflicts with other systems. Additionally, littlefs drivers that support
attributes should provide a "ignore attributes" flag to users in case attribute
conflicts do occur.

Attribute types prefixes with 0x0 and 0xf are currently reserved for future
standard attributes. Standard attributes will be added to this document in
that case.

Here's an example of non-standard time attribute:
```
(8 bits)  attribute type  = time       (0xc1)
(72 bits) time in seconds = 1506286115 (0x0059c81a23)

00000000: c1 23 1a c8 59 00                                .#..Y.
```

Here's an example of non-standard permissions attribute:
```
(8 bits)  attribute type  = permissions (0xc2)
(16 bits) permission bits = rw-rw-r--   (0x01b4)

00000000: c2 b4 01                                         ...
```

Here's what a dir entry may look like with these attributes:
```
(8 bits)   entry type       = file         (0x11)
(8 bits)   entry length     = 8 bytes      (0x08)
(8 bits)   attribute length = 9 bytes      (0x09)
(8 bits)   name length      = 12 bytes     (0x0c)
(8 bytes)  entry data       = 05 00 00 00 20 00 00 00
(8 bits)   attribute type   = time         (0xc1)
(72 bits)  time in seconds  = 1506286115   (0x0059c81a23)
(8 bits)   attribute type   = permissions  (0xc2)
(16 bits)  permission bits  = rw-rw-r--    (0x01b4)
(12 bytes) entry name       = smallavacado

00000000: 11 08 09 0c 05 00 00 00 20 00 00 00 c1 23 1a c8  ........ ....#..
00000010: 59 00 c2 b4 01 73 6d 61 6c 6c 61 76 61 63 61 64  Y....smallavacad
00000020: 6f                                               o
```
