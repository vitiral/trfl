# fngi filesystem

This is an in-development (currently conceptual) filesystem for the
[fngi](http://github.com/civboot/fngi) kernel, intended to be used
for a bootloader or on a microcontroller/etc, but robust enough as a general
purpose file system.

This is an extremely simple embedded filesystem with low memory requirements. It
stores a Binary Search Tree of directories, files and their data with the
ability to create, delete, write, append, read and modify.

## Basics

The filesystem is designed for SD cards, but I believe could have reasonable
performance for other media (i.e. floppy, spinning disk, etc) since the write
performance would never require seeking. However, because it's designed for SD
cards, it performs wear-leveling, only erasing each byte once before cycling
back to the beginning. It follows the block erasure principles required by SD
Cards, read about them
[on Wikipedia](https://en.wikipedia.org/wiki/Flash_memory#Block_erasure)

> One limitation of flash memory is that, although it can be read or programmed
> a byte or a word at a time in a random access fashion, **it can be erased only
> a block at a time**. This generally sets all bits in the block to 1. Starting
> with a freshly erased block, any location within that block can be programmed.
> However, once a bit has been set to 0, only by erasing the entire block can it
> be changed back to 1. In other words, flash memory (specifically NOR flash)
> offers random-access read and programming operations but does not offer
> arbitrary random-access rewrite or erase operations. A location can, however,
> be rewritten as long as the new value's 0 bits are a superset of the
> over-written values. For example, a nibble value may be erased to 1111, then
> written as 1110. Successive writes to that nibble can change it to 1010, then
> 0010, and finally 0000. Essentially, erasure sets all bits to 1, and
> programming can only clear bits to 0.[79] Some file systems designed for flash
> devices make use of this rewrite capability, for example Yaffs1, to represent
> sector metadata. Other flash file systems, such as YAFFS2, never make use of
> this "rewrite" capability—they do a lot of extra work to meet a "write once
> rule".

For reference, that wikipedia article also states the [typical block
sizes](https://en.wikipedia.org/wiki/Flash_memory#NAND_memories). The smallest
is 16KiB, the largest is 512KiB.

To take this into account, the following principles are followed:
- Every data structure starts with some flags/bits "reserved" and set to `1`
  ("erased"). They may later be set to 0 by the Garbage Collector (GC),
  indicating that data has moved/etc. More on the GC later.
  - Note: the limitation of flash memory allows any single _bit_ to be "cleared"
    to `0` at any time. It is setting it to `1` which requires a "block
    erasure".
- When data is "deleted" it simply has a single bit (flag) cleared. It will
  later be cleaned by the GC
- Data "Write Head": data is written sequentially from low-memory to high
  memory, then cycles back to low memory. Every 64KiB sector has metadata
  indicating which 64 byte "slots" have been used.
- At least 2+ "erasure blocks" in front of the Write Head is the Garbage
  Collector (GC). It finds non-deleted data and tells the write-head to write
  it, moving the data. Once all non-deleted data has been moved, it performs a
  block erasure and updates by setting (to `0`) the relevant reserved bits (i.e.
  reference pointers) of the parents of any moved data.
  - For example: if you have a binary search tree, and your left node has moved.
    You will have a flag indicating this fact, and you will have a "new"
    reference set aside for the new data.

## Limitations

- Not good for large non-mutating data. Large files especially will cause "GC
  Churn" as it moves the whole file a few erasure blocks to the left to allow
  space for the write head.
  - Alternative: simply partition the SD Card and put large long-lived files in
    the new partition and symlink them from your filesystem. A different
    filesystem designed only for large files (which can take up whole "erasure
    blocks") could easily be created to accommodate this need.
- Not necessarily good for frequently mutating files, especially large files.
  Each mutation requires the entire file to be re-written to the write head.
- Large numbers of renamings can slow down file and directory lookup in the
  directories where renamings have happened. The OS may occasionally need to
  "cleanup" the directory structure if this happens.

Some of these issues could be solved in the future. For example, the GC could
automatically move large files to a separate partition for large files when it
first encounters them. For now, these issues are adequate for the current
version.

## Data Structures

Note that data structures are in the filesystem and are therefore tightly packed
(have no alignment requirements).

```
\ A reference to another area in the SDCard. The number of bytes depends on
\ the size of the SDcard.
typedef Ref Arr[U1; sdcardSize]

\ A Ref that can be updated by the GC
struct GCRef {
  isFirst: U1,
  first: Ref,
  second: Ref,
}

struct Sector {
  flags: U1, \ <unused> | uninitialized | notRoot | firstGCRunning |
             \ finalGCRunning
  cycle: U8,  \ 8 byte cycle count
  root: GCRef,
  freeBitmap: Ref Array[U1; 64],
}
```

The sector contains a flag `notRoot` specifying whether this sector contains
the reference to the "root" node (i.e. `/` in a Linux filesystem). At filesystem
startup, a binary search (time `O(log n)`)is performed over all sectors to find
this sector. The binary search uses the `cycleCount`, which increments every
time the GC collects a sector. This also finds the current GC location, since it is a known number of
"erasure blocks" in front of the root sector. After this, the filesystem will
keep track of where the root node is stored.

All GCRef's contain a "second" reference to use when the first is Garbage
collected. A node will only be updated once, as the GC will wrap around to
clean it up (setting to the `first` reference) before it needs to be updated
again.

### Directories

A Node struct is:

```
\ Signed time in Milliseconds since/before epoch.
typedef TimeMilliseconds I8;

struct Owner {
  owner: U4,        \ user id
  group: U4,        \ group id
  permissions: U1,  \ i.e. linux chmod
}

struct Data {
  flags: U1, \ alive | uninitialized
  created: TimeMilliseconds,
  parent: GCRef,
  len: U2, \ 2 byte length, maximum size is 1 sector (64KiB)
  next: GCRef[Data], \ more data
  data: Arr[U1, len],
}

struct Node {
  flags: U1, \ alive | uninitialized
  created: TimeMilliseconds,
  owner: Owner,
  left: GCRef[Node],
  right: GCRef[Node],
  name: GCRef[Data]
  parent: GCRef[Node],
  variantEnum: dir | file | symlink | ...
  VariantData: Dir | File | ...
}
```

Data simply contains raw bytes, up to ~64KiB in size and the ability to update
it's parent's reference when it is moved by the GC.

The "Variant data" contain either a Dir (directory) or File (raw data). Later
there may also be symlinks, etc.

```
struct Dir {
  root: GCRef[Node],
}
```

A Dir is simply contains the root of a new Binary Search Tree (BST) node, which
may be "null" (all 1's). Note that the other data like name, flags (like `del`),
etc are contained in the `Node` struct.

#### Deletion, Re-naming and Cleanup
If a directory node needs to be renamed, it simply has `del` set and a new node
is created. The BST search algorithm will use it for finding nodes, but will
normally not return it. When the node is GC'd it will be removed. The OS can
also choose to rewrite the whole directory structure to permanently delete it
and all other nodes, which may sometimes be required if a directory contain too
many deleted nodes, especially deleted nodes with the same name.

When the directory structure is re-written, the OS may also reballance all
binary search trees for faster lookups.

### Files
Files can contain a lot of data and are sometimes updated very frequently
(citation needed). The file's _name_ is changed in the same way as directories
(it is a normal `Node` variant after all), however it's data uses a new
structure designed for frequent changes:

```
struct VersionedData {
  flags: U1, \ alive | uninitialized | isCurrent
  nextVersion: GCRef[VersionedData],
  len: U1,   \ number of versions stored in this struct
  dataIndex: Arr[U1; len/8],          \ Bitmap selecting which data field is correct
  dataArr:   Arr[GCRef[Data]; len],   \ References directly to data
}
```

When VersionedData needs to be changed, it simply updates the `dataIndex` bitmap
with the current version (setting the highest non-`1` bit to `0`). When this
`VersionedData` is completely used, it reserves a new one at `nextVersion` which
reserves twice the current `len` (with the limit that a VersionedData struct
can only be the size of one sector).  Because of this, the time to find which
version a node is therefore `O(log v)`, where v is the number of versions. When
a file is GC'd (or the entire directory structure is cleaned up, see the **Dir**
section) then only the latest version is copied.

#### Appending Data
Whether a file is being written to or a directory/file is being added to a
directory, all data is appended by first writing the entire piece of data,
including the parent reference and then updating the parent.

This means that if there is power-loss while the parent is being updated, the
startup procedure can detect this (by looking at the last `initialized` node
being written and it's parent) and correct the problem.

Appending to `Data` is simply updating the `next` field.  Similarly, (although
probably rarely) renaming a file/dir by adding to the name operates in the same
way.

Mutating data (non appended) requires rewriting the whole file and updating the
version. Mutating a file or directory name requires adding a new one and
clearing the `alive` bit.

When the GC runs on Data which has a `next` field, it will join appended
sections up to the current sector size.

## Write Head (WH)
Both the GC and data writing are designed to be **byte atomic**, meaning that
power can be lost at any time and when the system reboots it will still be
able to recover the filesystem (although some space may be lost on that cycle).

Data is written, filling up one sector at a time. The sector starts off as
"cleared" (all bits set to 1), and therefore has flags `uninitialized=true,
notRoot=true, firstGCRunning=true`. Likewise, the sector's `freeBitmap` is all set to
1, indicating all 64 byte slots are "free".

Basic operation:
- Before the sector is reserved, the GC has set `firstGCRunning=0`
- The sector is "reserved" by updating the current cycle, setting the current
  root and marking the slots used by the sector as not-free. `uninintialized` is
  then set to `0`, leaving `notRoot=1` (this sector is now root). The previous
  sector's `notRoot` is then set to 0.
  - Startup will check for the previous non-atomicity before selecting the root
    sector. In general, any non-atomicity is designed to be deterministic and
    therefore possible to check for.
- The WH then receives data to write equal to or less than the remaining size of
  the sector. The WH does not need to check types or do any other special
  operation. It simply updates it's `freeBitmap` as data is written.
  - Startup will check for non-agreement. If the final slot is marked as "free"
    but is not set to all `1`'s, then it will assume power loss and mark it as
    non-free.
  - Note: No references to data are set before that data has been written.

## The Garbage Collector (GC)
The GC runs more than 1 (probably 2 or more)  "block erasures" in front of the
WH, moving non-deleted data to the write head and erasing blocks. After all data
in a sector has been moved, it clears the `finalGCRunning` flag.

The GC knows how to determine the size of a node. It simply scans the sector for
living nodes, knowing whether they are by using their lengths.

## Data Corruption
TODO: Data corruption is a real problem for storage systems. This section should
detail how this will be solved for.

Possible Solutions:

1. CRC checksum for all non-mutable data (i.e. the `Data` struct), which can
   both find issues and fix them.
2. Double or even triple duplicated data for mutable data (i.e. flags, refs,
   etc).
   - Most bitmaps don't require duplication as their neighbor will account for any
     mis-setting. However, versioned bitmaps probably require
     double-duplication.
3. Bad-block bitmaps
4. Other solutions?

## LICENSING

This work is part of the Civboot project and therefore primarily exists for
educational purposes. Attribution to the authors and project is appreciated but
not necessary.

Therefore this body of work is licensed using the [UNLICENSE](./UNLICENSE),
unless otherwise specified at the beginning of the source file.

If for any reason the UNLICENSE is not valid in your jurisdiction or project,
this work can be singly or dual licensed at your discretion with the MIT
license below.

```text
Copyright 2022 Garrett Berg

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

