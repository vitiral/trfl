# fngi filesystem

This is an in-development (currently conceptual) filesystem for the
[fngi](http://github.com/civboot/fngi) kernel, intended to be used
for a bootloader or on a microcontroller/etc, but robust enough as a general
purpose file system.

This is an extremely simple embedded filesystem with low memory requirements. It
stores a Binary Search Tree of directories, files and their data with the
ability to create, delete, write, append, read and modify.

## Basics

**Stages of the Write Head (WH) and the Garbage Collector (GC) Head**
```

   Filesystem start, disk entirly free (` `)

     WH
     |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
     |---------------------------------------------------------------|
     |                                                               |
     |                                                               |
     |                                                               |
     |                                                               |
     |                                                               |
     |---------------------------------------------------------------|

   Filesystem start, some data used (`U`), some deleted (`.`) Write Head (WH)
   writes data from `left->right`, as received by the operating system (user).

             WH
     |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
     |---------------------------------------------------------------|
     |UUU.UUUU                                                       |
     |U.UUUUUU                                                       |
     |UUU.UUU                                                        |
     |UUUUUU.                                                        |
     |UUUUU.U                                                        |
     |---------------------------------------------------------------|

   Write Head (WH) nears end of SD Card and Garbage Collector (GC) Head starts.
   Note that much of initial data is now deleted (`.`) due to normal filesystem
   operation (moving files, changing contents, etc). GC begins also sending
   still used (U) data to the WH, which writes it alongside the new data from
   the OS (user). Whenever the GC moves data, it updates the parent references.
   Once all used data has been moved and references updated, the block can be
   freed (` `).

     GC -----------------------------------------------> WH
     |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
     |---------------------------------------------------------------|
     |U...U..UU...UUUU...UUUU...UUUU..UU..UUUU..UUUU..UUU            |
     |U.U......UUU...UUUU...UUUUUUUUUU..UUUU..U..UUUUUU..            |
     |..U.U...UU.U...UU...UUUUU...UU..UUUUU..UU..UUU..U.U            |
     |U.U.....U.UUUU...UUUUUUUUUUU...UUUUUU..UUU..UUU.U.U            |
     |U...U.UUUU...UUUU...UUUUUUUUUU...UUUUUU..UUUUUU.UU             |
     |---------------------------------------------------------------|

   GC has erased data and WH has wrapped around. GC continues to send used data
   to the WH and erase blocks. There is no memory fragmentation because any
   still used data is continuously re-packed by the write head on each cycle.

         WH <------- GC
     |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
     |---------------------------------------------------------------|
     |UUUU           U...U..U...UUUU..UU..UUUU..UUUU...UUU..UUU.U..UU|
     |U.U            U..U...U..U..UUUU..UUUU..U..U..U...UUU...UU.U.UU|
     |UUU            .U...U.U.....UU..UUU.U..UU...UU....U..UUU.UUU.UU|
     |UUU            ..UU....UUU.....UU..U...UUU..U.U.U.U..UUU..U..UU|
     |U.u            U....UU..UUUUUU...UU.U.U.........UU.U.UU...UU.UU|
     |---------------------------------------------------------------|

```

The filesystem is designed for SD cards, but I believe it could have reasonable
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
> programming can only clear bits to 0. [79] Some file systems designed for flash
> devices make use of this rewrite capability, for example Yaffs1, to represent
> sector metadata. Other flash file systems, such as YAFFS2, never make use of
> this "rewrite" capability—they do a lot of extra work to meet a "write once
> rule".

For reference, that wikipedia article also states the typical [Erasure Block]
sizes are between 16KiB and 512KiB.

> Quick note: In actual fact, the hardware may be implemented so that "erasure"
> sets the bits to `0` and bits can only be set to `1`. Since this is easier
> conceptually, it will be the mental model used and the physical layer will
> invert the data as needed. For more info see section 4.3.5.1 of the [Physical
> Layer Spec].

To take this into account, the following principles are followed:
- Every data structure starts with some flags/bits "reserved" and set to `0`
  ("erased"). They may later be set to 1 by the Garbage Collector (GC),
  indicating that data has moved/etc. More on the GC later.
  - Note: the limitation of flash memory allows any single _bit_ to be set
    to `1` at any time. Erasing to `0` requires a "block erasure" (these values
    may be reversed, but we use this mnemonic for simplicity).
- When data is "deleted" it simply has a single bit (flag) set. It will
  later be cleaned by the GC
- Data "Write Head": data is written sequentially from low-memory to high
  memory, then cycles back to low memory. Every 64KiB sector has metadata
  indicating which 64 byte "slots" have been used.
- At least 2+ "erasure blocks" in front of the Write Head is the Garbage
  Collector (GC). It finds non-deleted data and tells the write-head to write
  it, moving the data. Once all non-deleted data has been moved, it performs a
  block erasure and updates by setting the relevant reserved bits (i.e.
  reference pointers) of the parents of any moved data.
  - For example: if you have a binary search tree, and your left node has moved.
    You will have a flag indicating this fact, and you will have a "new"
    reference set aside for the new data.

## Core Data Structures

There are a few core data structures used throughout the design. Note that all
of these represent tightly packed data (with no alignment requirements) with
big-endian (aka network-endian, highest byte first) byte ordering.

### References
First we have the `Ref` and `GCRef`, which are methods to refer to other areas
of the filesystem at a certain byte address. `GCRef` is simply two `Ref`s, which
allows the GC to "move" data (Recall that data can only be set once, so in order
to move data you have to have the space set aside and a flag to indicate
movement).

```
\ A reference to another area in the SDCard. The number of bytes is always 5,
\ which is enough for 1 TiB of data.
typedef Ref Arr[U1; 5]

\ A Ref that can be updated by the GC
\ The flags contain the following bits:
\   del: if set, this ref is no longer active. Used for versioned data.
\   firstInit: if set, first data has been successfully initialized
\   secondInit: if set, second data has been successfully initialized
struct GCRef[A] {
  flags: U1, \ del | firstInit | secondInit | _unused_ ...
  first: Ref[A],
  second: Ref[A],
}
```

GCRef allows for changes to a reference by the GC, which is necessary because
the GC will move all data exactly once before collecting it. However, we need a
way to specify arbitrary changes to data (i.e. change to file contents), which
is what `VersionedRef` is used for.

### Versioned Data

The VersionedRef is simply an array of the past and (possibly future and
current) version of some data. If `next` is set, then look there for the current
version.  Otherwise, the last initialized version in `versions` the current one.

Why do we do this? Recall, we can only set bits, never clear them... unless we
clear a whole sector, which is what the GC does. We have to reserve space for
future changes ahead of time.

When `versions` are exhausted, a new set of versions with twice the length are
reserved at `next`. Therefore the time to find a version is the same as
performing a binary search: `O(log v)` where v is the number of versions.

```
struct VersionedRef[A] {
  next: GCRef[VersionedRef], \ if set, then use instead of these versions
  len: U2, \ the number of possible versions
  versions: Arr[GCRef[A], len], \ references to relevant data/nodes
}

\ A compressed version of VersionedRef with known size, used in other structs.
struct VersionedRefStart[A] {
  start: GCRef[A]
  next: GCRef[VersionedRef[A]],  \ if set, then use instead of start
}
```

### Ropes for Byte Data
Finally, we have data storage in the RawData and Rope structs. These are
referenced by `VersionedRef`'s above, which are themselves used in the `File`
object seen later.

> **Why Ropes?**: Ropes enable both fast appends _and_ fast and efficient
> mutation of data. Along with binary search trees they are pretty much the
> rockstars of filesystems.

```
struct RawData {
  flag: U1, \ del | initialized | type=RawData
  parent: GCRef,
  crc32: U4, \ CRC32 checksum on len + data
  len: U2,
  data: Arr[U1, len],
}

\ Versioned rope length
struct RopeLen {
  flag: U1, \ del | initialized | type=RopeLen
  parent: GCRef[RopeLen],
  next: GCRef[RopeLen], \ if set, use instead of this
  len: U2, \ number of versions
  lengthVersions: Arr[U2, len], \ length values at versions
}

struct Rope {
  flag: U1, \ del | initialized | type=Rope
  parent: GCRef[Node], \ Filesystem Node
  up:    VersionedRefStart[Rope | RawData | SectorData],
  left:  VersionedRefStart[Rope | RawData | SectorData],
  right: VersionedRefStart[Rope | RawData | SectorData],
  len: RopeLen, \ note: uses the remainder of the slot
}
```

To see how ropes work, let's look at an example. `R[len]` represents a rope node
and `"some string"` represents RawData.

```
  We start writing some data, which goes on the left of the Rope:

   /----------------R[12]
   |
 "hello world!"

  Now we want to append " And Hello Universe!" to it. To do so we add a new
  version to the right and update the Rope:

   /----------R[32] -----\
   |                     |
 "hello world!"  " And Hello Universe!"

 Now, we want to append " Now Goodbye!", we reserve an "up" node, which adds the
 data to its right:

               /--------------R[45]-----\
               |                        |
   /----------R[32] -----\         " Now Goodbye!"
   |                     |
 "hello world!"  " And Hello Universe!"

 Now, we wish to change "hello world" to "Hello Amazing World" since we love
 capitalizing amazing things. We simply update to a new version of the data and
 update the lengths:

               /--------------R[53]-----\
               |                        |
   /----------R[40] -----\         "Now Goodbye!"
   |                     |
 "Hello Amazing World!"  " And Hello Universe!"

 We decide against "Amazing" and want to remove it. If the data is small we
 simply replace the data. If the data is large, we could instead decide to split
 it in order to avoid future large mutations (shown here for demonstration):

               /--------------R[45]-----\
               |                        |
   /----------R[32] -----\         "Now Goodbye!"
   |                     |
 /-R[12]----\            |
 |          |            |
 "Hello "  "World!"  " And Hello Universe!"
```

A few notes on ropes:
- For a balanced rope, the time for most operations (seek, insert, etc) is `O(log v + log n)`.
  Streaming operations like streaming read/write is `O(1)` after the initial
  seek since it involves just traversing/building the tree from a known
  location.
- We can choose to re-balance the rope at any time, which will also reduce the
  number of versions. We simply combine the RawData's and create a new rope,
  then update the version of the parent node.
- Ropes can start out using small RawData chunks (say maximum ~500 bytes), then
  the GC can "graduate" them to larger chunks or even SectorData if they were
  not mutated during the cycle. The GC can tell they were not mutated by reading
  their version structs.

## Sector
The size of a sector is the size of an [Erasure Block]. It contains some
metadata at the beginning of every erasure block:

```
struct Sector {
  \ del | initialized | notRoot | firstGCDone | finalGCDone | type=Sector
  flags: U1,
  cycle: U4,  \ 4 byte cycle count
  root: GCRef,
  allocBitmap: Arr[U1, sectorLen / 8],
}
```

The sector contains a flag `notRoot` specifying whether this sector contains the
reference to the "root" node (i.e. `/` in a Linux filesystem). At filesystem
startup, a binary search (time `O(log n)`) is performed over all sectors to find
this sector. The binary search uses the `cycle` count, which increments every
time the GC collects a sector. This also finds the current GC location, since it
is a known number of "erasure blocks" in front of the root sector. After this,
the filesystem will keep track of where the root node is stored.

The allocBitmap has each bit set as the relevant 64 byte slot is used. This
keeps track of the currently allocated data for the event of power loss. This is
checked against any data which is written to determine if there was critical
power loss (corruption) and help in recovery.

### SectorData for large Blocks of Data

If the RawData would be nearly the size of a single sector/[Erasure Block], it
can be the type SectorData. This _replaces_ the normal sector metadata and has
type=SectorData.

```
struct SectorData {
  flags: U1, \ del | initialized | type=SectorData
  crc32: U4, \ CRC32 checksum of len + data
  len: U3,   \ space up to the remaining "Erasure Block" size.
  data: Arr[U1, len],
}
```

When the GC encounters SectorData which has not been deleted, it doesn't move
it. This drastically reduces churn on the GC for large files.

Since the data is not moved the GC does not need to update the parent reference;
therefore no parent reference is stored. Similarly, when the SectorData's parent
is moved, it doesn't need to update that fact since there is no reference to
update. This allows the sector data to essentially be "frozen" and reduces the
number of erasures on blocks where sector data is stored.

### Node Struct

The filesystem is composed of a binary search tree of "nodes" which have names
and associated data, which is a versioned reference to either a new node-tree or
data.

```
\ Signed time in Milliseconds since/before epoch.
typedef TimeMilliseconds I8;

struct Owner {
  owner: U4,        \ user id
  group: U4,        \ group id
  permissions: U1,  \ i.e. linux chmod
}

struct Node {
  created: TimeMilliseconds,
  owner: Owner,
  name: GCRef[RawData]
  left: GCRef[Node],
  right: GCRef[Node],
  parent: GCRef[Node],
  data: VersionedRefStart[Node | Rope],
}
```

#### Deletion, Re-naming and Cleanup
A node is considered deleted if all of its used versions have the `del` flag
set. If the node-name is ever re-used, then the new data will be added to the
VersionedRef. Note that a node's parent and left/right nodes _cannot_ be
updated by anyone except the GC, so even though the node is "deleted" it is
still used for the BST search until the GC collects it.

Renaming can be accomplished with a deletion then an add. Any data held by the
node must be re-written (unless that data is SectorData) since there is no way
to update the `Parent` reference.

The OS or user may also occasionally decide to rewrite a directory structure in
order to cleanup any large VersionedRef's or re-balance binary search trees,
however note that this is an extremely expensive operation requiring rewriting
almost all files (and sub-files) in a directory -- so this is only useful in
specific cases where there are an extremely large number of nodes in a
directory.

## Write Head (WH)
Both the GC and data writing are designed to be byte atomic, meaning that power
can be lost at any time and when the system reboots it will still be able to
recover the filesystem (although some space may be lost on that cycle).

Data is written, filling up one sector at a time. The sector starts off as
"cleared" (all bits set to 0), and therefore has flags `initialized=false,
notRoot=false, firstGCDone=false`. Likewise, the sector's `allocBitmap` is all
set to 0, indicating all 64 byte slots are "free".

Basic operation:
- Before the sector is reserved, the GC has set `firstGCDone=0`
- The sector is "reserved" by updating the current cycle, setting the current
  root and marking the slots used by the sector as allocated. `inintialized` is
  then set to `1`, leaving `notRoot=0` (this sector is now root). The previous
  sector's `notRoot` is then set to 1.
  - Startup will check for the previous non-atomicity before selecting the root
    sector. In general, any non-atomicity is designed to be deterministic and
    therefore possible to check for.
- The WH then receives data to write equal to or less than the remaining size of
  the sector. The WH does not need to check types or do any other special
  operation. It simply updates it's `freeBitmap` as data is written.
  - Startup will check for non-agreement. If the final slot is marked as "free"
    but is not set to all `0`'s, then it will assume power loss and mark it as
    non-free.

## The Garbage Collector (GC)
The GC runs more than 1 (probably 2 or more) sectors in front of the WH, moving
non-deleted data to the write head and erasing blocks. After all data in a
sector has been moved, it clears the `finalGCRunning` flag.

The GC knows how to determine the size of a node. It simply scans the sector for
non-deleted nodes, knowing where they are by using their lengths/etc.

## Data Corruption
TODO: Data corruption is a real problem for storage systems. This section should
detail how this will be solved for.

Possible Solutions:

1. CRC checksum for all non-mutable data (i.e. the `Data` structs), which can
   both find issues and fix them.
2. Double or even triple duplicated data for mutable data (i.e. flags, refs,
   etc).
3. Bad-block bitmaps
4. Other solutions?

## Physical Write Layer API
On most SDCards (i.e. non "Standard Capacity" cards) both reads and writes can
only be to a fixed 512 byte "block", which I'll call a block512 (see [Physical
Layer Spec] section 7.2). Also, from 7.1, SDUC cards are simply not supported in
SPI mode.

To accommodate this, but still allow byte-level control from an API standpoint,
the write interface will use a 512 byte buffer. To write to a byte at ref:

```
Block512 bl512 = asBlock512(ref);
readBlock512(&buf, bl512); // read 512 bytes of data
U2 offset = asOffset512(ref);
buf[offset] |= 0x84;  // set some bits (note that only set is allowed)
writeBlock512(&buf, bl512); // write 512 bytes of data.
```

Most loops/etc will work as if against a normal buffer but only actually write
data when the bl512 changes, then will call readBlock512 again.

For "erasing" data (setting to 0) you send `CMD32` then `CMD33` to set the
start/end address of the erasure. These will use `Block512` indexes. You then
send `CMD38:ERASE`.

Note in actual fact that erasure may set bits to 1. For those devices, all
communication will be inverted before sent. However, from the client's
perspective all operations will be "normal."

## Bibliography

- [Reversing CRC - Theory and
  Practice](http://stigge.org/martin/pub/SAR-PR-2006-05.pdf) is a complete
  description of CRC in 24 pages according to [this
  response](https://stackoverflow.com/a/4812571)
  - [rosseta code](https://rosettacode.org/wiki/CRC-32#C) example CRC-32 is only
    ~20 lines of C. It does require a 256 length U4, aka a **1024 byte table**.
- SD Card [Physical Layer Spec] for any specifics about SD Card SPI
  communication.

### Some notes

"Polynomial mapping" in the literature is fairly confusing. These two examples
should help explain what the literature is talking about.

Note that the literature writes the polynomial in "reverse" order of
most-significant bit first (i.e. it writes x^2 + 1 instead of 1 + x^2).
```
Repr     | Mapping                                             | Final
Poly     | 1^0 + 0^1 + x^2                                     | x^2 + 1
Binary   | 1     0     1                                       | 0xA

Poly     | 1^0 + 0^1 + 0^2 + x^3 + 0^4 + x^5 + 0^6 + 0^7 + x^8 | x^8 + x^5 + x^3 + 1
Binary   | 1     0     0     1     0     1     0     0     1   | 0x94
```

[Erasure Block]: https://en.wikipedia.org/wiki/Flash_memory#NAND_memories
[Physical Layer Spec]: https://www.sdcard.org/downloads/pls/

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

