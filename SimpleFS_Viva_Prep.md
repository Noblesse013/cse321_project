# SimpleFS Viva Preparation

CSE321 Lab Term Project — 60 marks

Read this in order. Part 1 builds the mental model from nothing. Part 2 walks
the code. Part 3 is the questions. Part 4 is the corner cases that separate a
good answer from a great one.

---

# PART 1 — The Mental Model

## 1.1 What a file system actually is

A disk is one enormous array of bytes. That is all it is. There are no files on
it, no names, no folders — just bytes numbered 0, 1, 2, 3, and so on.

A file system is an **agreement about what those bytes mean**. If everyone
agrees that "the first 4096 bytes describe the layout" and "bytes 16384 onward
hold file contents", then a program can open that array of bytes and reconstruct
files from it. Change the agreement and the same bytes mean something else.

SimpleFS is a small, written-down version of that agreement.

**Say it like this in the viva:** "The image is just a byte array. The file
system is the set of rules for interpreting it."

## 1.2 Why blocks

Tracking every individual byte would need an enormous amount of bookkeeping —
one entry per byte. Instead the disk is divided into fixed-size chunks called
**blocks**, and everything is tracked per block.

In SimpleFS a block is 4096 bytes. There are 64 of them. So:

    64 blocks x 4096 bytes = 262144 bytes = 256 KiB

That is why the image is exactly 262144 bytes, and why the first test case
checks that number.

The cost of blocks: a 1-byte file still uses a whole 4096-byte block. This is
called **internal fragmentation**. The benefit: bookkeeping is 4096 times
smaller. Every real file system makes this trade.

## 1.3 Finding a block

Because blocks are fixed size, the position of any block is pure arithmetic:

    byte offset of block N = N x 4096

Block 0 starts at byte 0. Block 1 at 4096. Block 4 at 16384. There is no lookup
table, no searching. This is the main reason block sizes are fixed.

In C this is what `fseek` does:

```c
fseek(fp, 4 * BLOCK_SIZE, SEEK_SET);   /* move to the start of block 4 */
```

`SEEK_SET` means "measured from the beginning of the file".

## 1.4 The five regions

SimpleFS divides its 64 blocks into five regions:

| Block | Region | Byte offset | What it holds |
|---|---|---|---|
| 0 | Superblock | 0 | Where everything else is |
| 1 | Inode bitmap | 4096 | Which inodes are in use |
| 2 | Data bitmap | 8192 | Which data blocks are in use |
| 3 | Inode table | 12288 | 32 inodes, 128 bytes each |
| 4–63 | Data region | 16384+ | Actual file contents |

Blocks 0 through 3 are **metadata** — data about data. Blocks 4 through 63 are
the only place real file contents live.

**Memorise the four offsets: 0, 4096, 8192, 12288, 16384.** Every question about
"where is X" is answered from those.

## 1.5 The superblock — the map

The superblock is a small struct at the very start of the image that records the
layout:

```c
typedef struct {
    uint32_t magic;               /* 0x53465331 - "is this a SimpleFS image?" */
    uint32_t block_size;          /* 4096 */
    uint32_t total_blocks;        /* 64 */
    uint32_t inode_count;         /* 32 */
    uint32_t inode_bitmap_block;  /* 1 */
    uint32_t data_bitmap_block;   /* 2 */
    uint32_t inode_table_block;   /* 3 */
    uint32_t data_region_block;   /* 4 */
    uint32_t root_inode;          /* 1 */
} superblock_t;
```

Nine fields, four bytes each = **36 bytes**. But the whole of Block 0 (4096
bytes) is reserved for it, and bytes 36 to 4095 must stay zero.

**Why reserve a whole block for 36 bytes?** Because everything is addressed in
blocks. If the superblock used only 36 bytes, the inode bitmap would start at
byte 36, and every later region would be at an awkward offset. Reserving whole
blocks keeps all the arithmetic clean, and leaves room to add fields later
without moving anything.

**Why a magic number?** It is a signature. When `simplefs_adder` opens a file, it
reads the first four bytes and checks them against `0x53465331`. If they do not
match, this is not a SimpleFS image — maybe it is a JPEG, maybe a text file, maybe
a corrupted image. Without that check the adder would happily write file data into
a random file and destroy it.

`0x53465331` is not random: `53` is 'S', `46` is 'F', `53` is 'S', `31` is '1' in
ASCII. It spells **SFS1**. That is why the dump shows `1SFS` in the ASCII column.

## 1.6 The endianness question (guaranteed to be asked)

You write `0x53465331`. The hex dump shows `31 53 46 53`. The bytes look
backwards. Why?

x86 processors are **little-endian**: a multi-byte number is stored with its
*least significant* byte first.

    0x53465331
      53 46 53 31     <- how you write it (most significant first)
      ^most        ^least

    In memory: 31 53 46 53
               ^least       ^most

So byte 0 of the file holds `0x31`, byte 1 holds `0x53`, and so on. The dump
prints bytes in file order, which is why it reads `31 53 46 53`.

**Nothing is wrong.** The number is stored correctly; `xxd` just shows you raw
byte order rather than numeric order.

The same explains every other field:

    block_size 4096 = 0x00001000  ->  dump shows: 00 10 00 00
    total_blocks 64 = 0x00000040  ->  dump shows: 40 00 00 00
    inode_count  32 = 0x00000020  ->  dump shows: 20 00 00 00
    root size   128 = 0x00000080  ->  dump shows: 80 00 00 00
    file size  5000 = 0x00001388  ->  dump shows: 88 13 00 00

Practise converting these both ways. It is the single most likely thing you will
be asked to do at the whiteboard.

## 1.7 The inode — metadata without a name

An **inode** ("index node") holds everything about a file except its name:

```c
typedef struct {
    uint16_t type;          /* 1 = file, 2 = directory */
    uint16_t links;         /* how many names point to this inode */
    uint32_t size;          /* actual length in bytes */
    uint32_t direct[3];     /* absolute block numbers of the data */
    uint8_t  reserved[108]; /* padding, kept zero */
} inode_t;
```

2 + 2 + 4 + 12 + 108 = **128 bytes exactly**.

**Why exactly 128?** So the inode table fits perfectly:

    32 inodes x 128 bytes = 4096 bytes = exactly one block

And so any inode can be found by arithmetic instead of searching:

    byte offset of inode I = 12288 + (I - 1) x 128

The `reserved[108]` field exists only to make the number come out at 128. Real
file systems use that space for timestamps, owner IDs, and permissions —
SimpleFS does not implement those, so it is padding.

**Why does the inode not store the name?** This is the deepest question in the
project, and worth a proper answer:

Separating names from metadata means **many names can refer to one file**. That
is what `links` counts. In a real file system you can have `/home/a.txt` and
`/backup/a.txt` both pointing at the same inode — one file, two names, link count
2. If the name lived inside the inode, that would be impossible.

It also means renaming a file touches only the directory entry, not the file
itself.

## 1.8 The direct pointers

`direct[0]`, `direct[1]`, `direct[2]` hold the **absolute block numbers** where
the file's contents live. Not offsets, not bitmap indexes — block numbers.

    direct[0] = 5   ->  first 4096 bytes of the file are in block 5
    direct[1] = 6   ->  next 4096 bytes are in block 6
    direct[2] = 0   ->  unused

**Why does 0 mean unused?** Because block 0 is the superblock. No file's data can
ever be in block 0, so 0 is free to serve as the "nothing here" marker. The same
trick appears in the directory, where `inode_no == 0` means an empty slot, because
inode numbers start at 1.

**Why is the maximum file size 12288 bytes?**

    3 direct pointers x 4096 bytes per block = 12288 bytes

There is nowhere to record a fourth block. Real file systems solve this with
**indirect pointers**: a pointer to a block that itself contains a list of block
numbers. A 4096-byte block holds 1024 four-byte numbers, so one indirect pointer
would add 1024 more blocks. SimpleFS deliberately leaves this out.

## 1.9 The bitmaps — one bit per object

How do you know whether inode 7 is free? You could scan every inode, but that is
slow. Instead there is a **bitmap**: one bit per object, 0 = free, 1 = used.

    Inode bitmap (Block 1):
      bit 0 -> inode 1
      bit 1 -> inode 2
      ...
      bit 31 -> inode 32

    Data bitmap (Block 2):
      bit 0 -> absolute block 4
      bit 1 -> absolute block 5
      bit 2 -> absolute block 6
      ...

32 inodes need 32 bits = 4 bytes. 60 data blocks need 60 bits = 8 bytes. Yet each
bitmap gets a full 4096-byte block, for the same reason the superblock does:
block-aligned regions keep the arithmetic simple, and there is room to grow.

**The single most confusing part of this project** is that data bitmap index 0
means block **4**, not block 0:

    absolute block = bitmap index + DATA_REGION_BLOCK   (index + 4)
    bitmap index   = absolute block - DATA_REGION_BLOCK (block - 4)

**Why?** The bitmap only tracks the data region. Blocks 0–3 are the superblock,
the two bitmaps, and the inode table. They are permanently in use and can never
be allocated to a file, so spending bits on them would be pointless. The bitmap
starts counting where allocation becomes possible.

## 1.10 Reading a bitmap byte

A byte holds 8 bits. `xxd` shows bytes in hex. To read a bitmap you convert hex
to binary, then read **right to left**, because bit 0 is the least significant
bit.

    hex 01 = binary 00000001 -> bit 0 set              -> inode 1 only
    hex 03 = binary 00000011 -> bits 0,1               -> inodes 1,2
    hex 07 = binary 00000111 -> bits 0,1,2             -> inodes 1,2,3
    hex 0f = binary 00001111 -> bits 0,1,2,3           -> inodes 1,2,3,4
    hex 1f = binary 00011111 -> bits 0,1,2,3,4         -> inodes 1..5
    hex ff = binary 11111111 -> all 8                  -> inodes 1..8

The same values in the data bitmap mean blocks 4, then 4-5, then 4-6, then 4-7.

**The two bit operations:**

```c
/* mark index as used */
bitmap[index / 8] |= (1 << (index % 8));

/* test whether index is used */
if (bitmap[index / 8] & (1 << (index % 8))) { /* used */ }
```

`index / 8` picks the byte; `index % 8` picks the bit inside it. Index 10 is
byte 1, bit 2. `1 << 2` builds the mask `00000100`. OR sets that bit and leaves
the rest alone; AND tests it.

## 1.11 The directory — where names live

A directory is not a special kind of object. It is **an ordinary file whose
contents happen to be a list of name-to-inode mappings**. The root directory has
an inode (number 1) and a data block (number 4) just like any file.

```c
typedef struct {
    uint32_t inode_no;   /* which inode; 0 means the slot is empty */
    uint8_t  type;       /* 1 = file, 2 = directory */
    char     name[59];   /* the name, null-terminated */
} dirent_t;
```

4 + 1 + 59 = **64 bytes exactly**.

    4096 bytes per block / 64 bytes per entry = 64 entries per block

**Why is the name limit 58 and not 59?** The field is 59 bytes, but a C string
needs a null terminator. 58 characters + 1 null = 59. A 59-character name would
leave no room for the terminator, and `strcmp` would read past the end of the
field.

**How many files can SimpleFS hold?**

    64 entry slots - 2 (for "." and "..") = 62 possible names

But there are only 32 inodes, and inode 1 is the root. So the real limit is
**31 files**. The directory block is never the constraint — the inode table is.

## 1.12 The root directory and "." and ".."

When the image is formatted, the root directory already contains two entries:

| Slot | Offset | inode_no | type | name |
|---|---|---|---|---|
| 0 | 16384 | 1 | 2 | `.` |
| 1 | 16448 | 1 | 2 | `..` |

`.` means "this directory" — it points to inode 1, which is the root itself.

`..` means "the parent directory" — but the root has no parent. It is the top.
So by convention `..` in the root points back at the root: inode 1 again.

That is why the root inode has `links = 2`: two names (`.` and `..`) refer to
inode 1.

And why `size = 128`: two entries x 64 bytes.

Each new file adds one entry, so the root size goes 128 -> 192 -> 256 -> 320.
The first user file lands in slot **2**, at byte offset:

    16384 + 2 x 64 = 16512

## 1.13 First-fit allocation

When a new inode or block is needed, scan from the start and take the first free
one. Stop looking immediately.

That is the entire algorithm. It is called **first-fit** because it fits the
request into the first available space.

**Alternatives worth knowing** (in case the examiner asks what else exists):

- **Best-fit** — scan everything, pick the smallest space that fits. Less waste,
  much slower, and meaningless here because all blocks are the same size.
- **Next-fit** — like first-fit but resume from where the last search stopped.
  Spreads allocations out.
- **Worst-fit** — pick the largest free space.

For fixed-size blocks, first-fit is the sensible choice: every block is
interchangeable, so there is no reason to look further than the first free one.

---

# PART 2 — Walking the Code

## 2.1 simplefs_builder, step by step

1. Parse `--image <name>` from the command line.
2. Create the file and write 64 zeroed blocks — this is what makes the image
   exactly 262144 bytes, and guarantees every unused byte is 0.
3. Fill the superblock struct and write it at offset 0.
4. Set bit 0 of the inode bitmap (inode 1 = root), write it at offset 4096.
5. Set bit 0 of the data bitmap (block 4 = root's data), write at offset 8192.
6. Build the root inode (type 2, links 2, size 128, direct[0] = 4) and write it
   at offset 12288.
7. Build the `.` and `..` entries and write them at offset 16384.
8. Close the file.

**Why zero-fill first?** Three reasons. It sets the file's size to exactly
262144 bytes. It guarantees unused inodes, unused bitmap bits, and unused
directory slots all read as 0 — which is what "free" means everywhere in this
design. And it means no leftover data from whatever was previously on disk can
leak into the image.

## 2.2 simplefs_adder, step by step

1. Open the image read/write (`"rb+"`) — read *and* write, without truncating.
2. Read the superblock, check the magic number. Refuse if it does not match.
3. Open the source file read-only. It must never be modified.
4. Measure its size: seek to the end, `ftell`, rewind.
5. Reject if larger than 12288 bytes.
6. Reject if the name is longer than 58 characters.
7. Compute how many blocks are needed.
8. Scan the root directory; reject a duplicate name.
9. Read the inode bitmap; find the first free inode.
10. Read the data bitmap; allocate blocks first-fit.
11. Copy the file contents into those blocks.
12. Write the new inode.
13. Write both bitmaps back.
14. Write the directory entry.
15. Increase the root inode's size by 64.
16. Close both files.

**Why are the checks (steps 5, 6, 8) before the allocations (9, 10)?** Because
rejecting a file *after* marking blocks as used would leave those blocks
allocated to nothing — a **leak**. Space would be lost until the image was
rebuilt. Validate first, then commit.

## 2.3 The two subtle bits

**Setting the bitmap bit inside the allocation loop.**

```c
for (int i = 0; i < required_blocks; i++) {
    int blk = find_free_data_block(data_bitmap);   /* first-fit */
    if (blk == -1) { /* error out */ }
    allocated_blocks[i] = blk;
    set_bit(data_bitmap, data_bitmap_index(blk));  /* MUST be here */
}
```

`find_free_data_block` returns the first block whose bit is clear. If the bit is
not set immediately, the next call sees the same clear bit and returns the same
block again. A three-block file would get block 5, block 5, block 5 — each write
overwriting the last, and the file would be corrupt.

Setting the bit inside the loop is what makes each iteration see the effect of
the previous one.

**Re-zeroing the copy buffer each iteration.**

```c
for (int i = 0; i < required_blocks; i++) {
    unsigned char buf[BLOCK_SIZE];
    memset(buf, 0, BLOCK_SIZE);                    /* MUST be here */
    fread(buf, 1, BLOCK_SIZE, source);
    fseek(image, (long)allocated_blocks[i] * BLOCK_SIZE, SEEK_SET);
    fwrite(buf, BLOCK_SIZE, 1, image);             /* full block always */
}
```

Take a 5000-byte file. The first pass reads 4096 bytes and writes them to block
5 — the buffer is full, nothing is left over. The second pass reads only 904
bytes. Without the `memset`, bytes 904 to 4095 of the buffer still hold the
*previous* block's contents, and those stale bytes get written into block 6.

With the `memset`, they are zero. That is what "the unused part of the last block
stays zero" means, and it is why the full 4096 bytes are always written rather
than just the bytes read.

## 2.4 The number conversions

Four different numbers, all small integers, all easy to confuse:

| Name | Range | Example |
|---|---|---|
| Inode number | 1–32 | inode 2 |
| Inode table index | 0–31 | index 1 |
| Data bitmap index | 0–59 | index 1 |
| Absolute block number | 0–63 | block 5 |

    inode table index = inode number - 1
    absolute block    = data bitmap index + 4

`find_free_inode` returns an inode **number** (index + 1) because the caller
passes it to `inode_offset()`, which subtracts 1 internally. `set_bit` needs an
**index**, so `free_inode - 1` is passed there.

`find_free_data_block` returns an **absolute block number** (index + 4) because
that value goes straight into `direct[]` and gets multiplied by 4096 to seek.

Getting either direction wrong writes data 128 or 4096 bytes away from where it
should be, silently.

---

# PART 3 — Likely Viva Questions

## Basics

**Q: What is a file system?**
An agreement about how to interpret a flat array of bytes as files, names, and
free space.

**Q: Why is the image exactly 262144 bytes?**
64 blocks x 4096 bytes.

**Q: What is in Block 0?**
The superblock — the record of where every other region lives.

**Q: Why does the superblock use only 36 of its 4096 bytes?**
Because addressing is done in whole blocks. Keeping every region block-aligned
makes offsets pure arithmetic, and leaves room for future fields.

**Q: What is the magic number for?**
To verify the file really is a SimpleFS image before writing to it. Without it
the adder could corrupt an unrelated file.

**Q: Why does the dump show `31 53 46 53` instead of `53 46 53 31`?**
Little-endian byte order: the least significant byte is stored first.

## Inodes

**Q: What does an inode store?**
Type, link count, size in bytes, and up to three data block numbers. Not the name.

**Q: Why not the name?**
So several names can refer to one file — that is what `links` counts. It also
makes renaming a directory-only operation.

**Q: Why is an inode exactly 128 bytes?**
So 32 of them fill exactly one 4096-byte block, and so any inode's position can
be computed rather than searched for.

**Q: Where is inode 2 stored?**
`12288 + (2 - 1) x 128 = 12416`.

**Q: What does `direct[2] = 0` mean?**
Unused. Block 0 is the superblock, so no file's data can be there, which frees 0
to act as a sentinel.

**Q: Why is the maximum file size 12288 bytes?**
Three direct pointers x 4096 bytes. There is no way to record a fourth block.

**Q: How would you support larger files?**
Add an indirect pointer: a pointer to a block that holds a list of block
numbers. One 4096-byte block holds 1024 four-byte entries, adding 1024 blocks
of capacity.

## Bitmaps

**Q: What does the inode bitmap do?**
One bit per inode; 0 free, 1 used. It lets allocation check availability without
reading the inode table.

**Q: The data bitmap's first byte is `07`. What does that mean?**
`07` is binary `00000111` — bits 0, 1, 2 set, which are blocks 4, 5, and 6.

**Q: Why does bitmap index 0 mean block 4?**
The bitmap only tracks the data region, which begins at block 4. Blocks 0–3 are
metadata and can never be allocated.

**Q: Why reserve a whole block for 8 bytes of meaningful bits?**
Block alignment, and room to grow if the disk were larger.

**Q: Explain `bitmap[index / 8] |= (1 << (index % 8));`**
`index / 8` selects the byte, `index % 8` selects the bit within it, the shift
builds a one-bit mask, and OR sets that bit while leaving the other seven alone.

## Directories

**Q: How does a name become an inode number?**
By scanning the root directory block for an entry whose `name` matches, then
reading its `inode_no`.

**Q: How big is a directory entry, and why?**
64 bytes: 4 for the inode number, 1 for the type, 59 for the name. 4096 / 64
gives exactly 64 entries per block.

**Q: Why is the name limit 58 characters?**
The 59-byte field must also hold the null terminator.

**Q: Why do both `.` and `..` point to inode 1?**
`.` is the directory itself. `..` is the parent, but the root has no parent, so
by convention it points to itself.

**Q: Why is the root's link count 2?**
Two names — `.` and `..` — refer to inode 1.

**Q: Why does the root's size start at 128?**
Two entries x 64 bytes.

**Q: Where does the first user file's directory entry go?**
Slot 2, at byte `16384 + 2 x 64 = 16512`. Slots 0 and 1 are `.` and `..`.

**Q: Why does the free-slot search start at index 2?**
So `.` and `..` are never overwritten. If they were, the directory would stop
describing itself.

**Q: Why does `inode_no == 0` mark a slot as empty?**
Inode numbers start at 1, so 0 is available as a sentinel — and the image is
zero-filled at creation, so every unused slot naturally reads as empty.

## Allocation

**Q: What is first-fit?**
Scan from the beginning and take the first free entry. Stop there.

**Q: Where is first-fit implemented?**
In `find_free_inode` and `find_free_data_block` — both loop from the start and
return on the first clear bit.

**Q: Why does `find_free_inode` start at index 1?**
Index 0 is inode 1, the root directory, which is permanently allocated.

**Q: Why must the bitmap bit be set inside the allocation loop?**
Otherwise the next first-fit call finds the same clear bit and returns the same
block again.

**Q: A 5000-byte file uses two blocks. What is in the `size` field?**
5000. `size` is the file's real length; the extra 3192 bytes of block 6 are
padding, not content.

**Q: Why write the full 4096 bytes even when only 904 were read?**
Because the block is being replaced wholesale. Combined with the zeroed buffer,
this guarantees the unused tail is zero rather than stale data.

## Verification

**Q: How did you verify your implementation?**
By dumping the image with `xxd` at each region's offset and checking the bytes
against the expected values — not by trusting that the program ran without
crashing.

**Q: What does `xxd -s 8192 -l 8 disk.img` show?**
The first 8 bytes of Block 2, the data bitmap.

---

# PART 4 — Corner Cases

These are what an examiner uses to separate students who understand the design
from those who memorised the happy path.

## 4.1 A zero-byte file

Needs **zero** data blocks. It still gets an inode (type file, links 1, size 0,
all three direct pointers 0) and a directory entry. The plain ceiling-division
formula gives `(0 + 4095) / 4096 = 0`, so it works out naturally — but the code
guards it explicitly anyway.

The file exists and has a name; it simply has no contents.

## 4.2 A file of exactly 4096 bytes

Exactly one block, not two. `(4096 + 4095) / 4096 = 1`. The classic off-by-one
here is using `size / BLOCK_SIZE + 1`, which would wrongly allocate two.

## 4.3 A file of exactly 12288 bytes

The maximum. Three blocks, all three direct pointers used, and `size = 12288`.
Accepted. 12289 bytes is rejected because there is no fourth pointer.

## 4.4 A name of exactly 58 characters

Accepted — 58 characters plus a null terminator fills the 59-byte field exactly.
59 characters is rejected. Testing both sides of that boundary matters: rejecting
58 would be just as wrong as accepting 59.

## 4.5 Filling the file system

Two different limits, and they are hit at different times:

- **Inodes run out first in most cases.** 32 inodes, 1 reserved for the root, so
  31 files maximum. `find_free_inode` returns -1 and the adder reports "no free
  inode available".
- **Blocks can run out sooner with large files.** 60 data blocks, 1 used by the
  root, leaving 59. Five 12288-byte files consume 15 blocks; twenty of them would
  exhaust the region long before the inodes.

So the binding constraint depends on file sizes. Many small files exhaust inodes;
few large files exhaust blocks.

## 4.6 What if a file is rejected after blocks were allocated?

Those blocks would be marked used in the bitmap but referenced by no inode — a
**space leak**. Nothing could ever reclaim them, because there is no deletion.
This is why every validation check happens before any allocation.

## 4.7 Why must the same file not be added twice?

Two directory entries with the same name would make lookup ambiguous — a search
would find whichever came first, and the second file's blocks would be
unreachable. Hence the duplicate check.

Note it compares **names**, not contents. Two files with identical contents but
different names are perfectly fine and get separate inodes and separate blocks.

## 4.8 Are the blocks of a file always contiguous?

Not necessarily. First-fit takes whatever is free. On a fresh image the blocks
happen to come out consecutive (5, 6, 7), but if the file system had holes, a
file could get blocks 5, 9, and 14. The `direct[]` array is exactly what makes
this work — the blocks are recorded individually, so they need not be adjacent.

This is worth saying out loud: it shows you understand *why* direct pointers
exist rather than just what they contain.

## 4.9 What happens after `find_free_directory_entry` runs?

It leaves the file position wherever its loop stopped. Every subsequent read or
write must `fseek` first. This is a general C-file-I/O point: the position is
shared state, and functions that read from a file move it as a side effect.

## 4.10 Why is the image opened `"rb+"` and not `"wb+"`?

`"wb+"` **truncates** the file to zero length on open. That would destroy the
image the adder is supposed to modify. `"rb+"` opens for reading and writing
while preserving the existing contents.

Meanwhile the source file is opened `"rb"` — read-only — because it must not be
modified. That is verifiable with `md5sum` before and after.

## 4.11 What is the `reserved[108]` field for?

Padding, to make the inode exactly 128 bytes. In a real file system it would
hold timestamps, owner and group IDs, and permission bits. SimpleFS does not
implement those, so it is kept zero — and the space is there if the design were
extended.

## 4.12 What is not implemented, and what would it take?

| Missing | What it would need |
|---|---|
| Deletion | Clear the directory entry, clear the bitmap bits for the inode and its blocks |
| Subdirectories | Allow a directory inode to own a data block of entries; recursive path lookup |
| Larger files | Indirect block pointers |
| Renaming | Change the name in the directory entry only — the inode is untouched |
| Hard links | Add a second directory entry pointing to the same inode, increment `links` |
| Permissions | Use part of `reserved[]` for mode and owner bits |

## 4.13 The classic mistakes (from the spec's own list)

1. Confusing bitmap index with absolute block number.
2. Confusing inode number with table index.
3. Allocating without setting the bitmap bit.
4. Forgetting to fill `direct[]`.
5. Forgetting the directory entry — the inode alone gives the file no name.
6. Storing a bitmap index in `direct[]` instead of an absolute block number.
7. Storing allocated size instead of actual size.
8. Not zero-filling the final block.

Be ready to say what each one would look like if it happened — the examiner may
describe a symptom and ask which mistake caused it.

---

# PART 5 — The Exercise That Predicts Your Grade

Get someone to hand you a hex dump of an image with two files in it, and answer
from the bytes alone:

1. Is this a valid SimpleFS image? (check bytes 0–3)
2. How many inodes are allocated? (byte at 4096, converted to binary)
3. How many data blocks are in use? (byte at 8192)
4. What are the file names? (ASCII at 16384 onward)
5. Which inode does each name map to?
6. How big is each file? (bytes 4–7 of its inode)
7. Which blocks hold each file's data? (bytes 8–19 of its inode)
8. What is the root directory's size, and does it match the number of entries?

If you can do all eight without help, you are ready.

## The five answers to have word-perfect

1. **`31 53 46 53`** — little-endian; the least significant byte of `0x53465331`
   is stored first.
2. **`links = 2`, `size = 128`** — `.` and `..` both name inode 1; two 64-byte
   entries.
3. **Bit 0 means block 4** — the bitmap tracks only the data region, which starts
   at block 4.
4. **Bit set inside the loop** — otherwise first-fit returns the same block
   every time.
5. **`size = 5000`, not 8192** — size is the real length; the rest of the second
   block is zero padding.
