# SimpleFS — Project Tracker

CSE321 Lab Term Project, Summer 2026
**Deadline: 28 August 2026, 11:50 PM**

> This is our internal working tracker, not the README we submit.
> The submission README is a separate file — see the bottom of this page.

---

## Status at a glance

| Track | Owner | TODOs | Status |
|---|---|---|---|
| Builder | Member A | Builder 1–6 | **Done, verified** |
| Adder — directory | Member A | Adder 3, 4, 10, 11 | **Done, untested** |
| Adder — allocation & data | Member B | Adder 1, 2, 5, 6, 7, 8, 9 | Not started |
| Extra error checks | Member B | (2 checks, see below) | Not started |
| Full test sweep | Member B | — | Blocked on above |
| Submission README | Member A | — | Not started |

---

## DONE — Member A

### `simplefs_builder.c` — complete and verified

| TODO | What it does | Verified by |
|---|---|---|
| 1 | Fill all nine superblock fields | `xxd -l 48 disk.img` → `3153 4653 0010 0000 4000 0000 2000 0000` |
| 2 | Mark inode 1 allocated (bitmap index 0) | `xxd -s 4096 -l 8 disk.img` → first byte `01` |
| 3 | Mark root data block allocated (index 0 = block 4) | `xxd -s 8192 -l 8 disk.img` → first byte `01` |
| 4 | Root inode: type 2, links 2, size 128, direct[0]=4 | `xxd -s 12288 -l 16 disk.img` → `0200 0200 8000 0000 0400 0000` |
| 5 | The `.` entry | `xxd -s 16384 -l 128` → `0100 0000 022e` at 16384 |
| 6 | The `..` entry | same dump → `0100 0000 022e 2e00` at 16448 |

Image is exactly 262144 bytes. Compiles clean under `-Wall -Wextra -std=c11`.

### `simplefs_adder.c` — four TODOs written, not yet testable

| TODO | Function | What it does |
|---|---|---|
| 3 | `filename_exists` | Scans all 64 root entries for a duplicate name |
| 4 | `find_free_directory_entry` | Seeks past `.` and `..`, returns first slot with `inode_no == 0` |
| 10 | (in `main`) | Builds the directory entry: inode number, type, name bounded to 58 chars |
| 11 | (in `main`) | `root_inode.size += sizeof(dirent_t)` |

These cannot run until Member B's allocation code exists — the adder exits before
reaching them.

---

## TO DO — Member B

All in `simplefs_adder.c`. **Do not edit TODO 3, 4, 10, or 11** — those are done.

### The seven TODOs

| TODO | Where | What it needs |
|---|---|---|
| 1 | `find_free_inode` | Loop indexes 1–31 (skip 0, that's the root). Return `index + 1` — an inode **number**, not an index. `-1` if none free. |
| 2 | `find_free_data_block` | Loop indexes 0–59. Return `index + DATA_REGION_BLOCK` — an **absolute block number**. `-1` if none free. |
| 5 | `main` | `required_blocks = (file_size + BLOCK_SIZE - 1) / BLOCK_SIZE;` plus a special case: 0 bytes → 0 blocks. |
| 6 | `main` | Loop `required_blocks` times: allocate, store in `allocated_blocks[i]`, and **set the bitmap bit inside the loop**. Error out on `-1`. |
| 7 | `main` | Per block: zeroed 4096-byte buffer (fresh each iteration), `fread` from source, seek to `block * 4096`, write the **full** 4096 bytes. |
| 8 | `main` | New inode: type file, links 1, `size = file_size` (real size, not blocks × 4096), `direct[i] = allocated_blocks[i]`. |
| 9 | `main` | `set_bit(inode_bitmap, free_inode - 1)` — note the `- 1`. |

### Two error checks the skeleton is missing

1. **Filename longer than 58 characters** — add near the top of `main`, *before* anything
   is allocated. Rejecting after allocation leaks blocks in the image.
2. **Insufficient free data blocks** — inside the TODO 6 loop, when
   `find_free_data_block` returns `-1`.

### Three traps that cost the most time

- **Bitmap bit inside the loop (TODO 6).** Allocate all three blocks first and set the
  bits afterwards, and first-fit hands you block 5 three times.
- **Re-zero the buffer each iteration (TODO 7).** This is what leaves the tail of the
  final block clean. A 5000-byte file writes 904 real bytes + 3192 zeros into block 6.
- **The ±1 on inode numbers (TODO 1 and 9).** `find_free_inode` returns a *number*;
  `set_bit` takes an *index*. Get the direction wrong and every file's inode lands
  128 bytes off.

---

## Test sweep — run after Member B finishes

Always start from a fresh image. **Recompile after every edit** — editing the `.c` does
not change the binary.

```bash
gcc -Wall -Wextra -std=c11 simplefs_builder.c -o simplefs_builder
gcc -Wall -Wextra -std=c11 simplefs_adder.c   -o simplefs_adder
```

### 1. One small file

```bash
rm -f disk.img && ./simplefs_builder --image disk.img
md5sum test1.txt                       # note this
./simplefs_adder --input disk.img --file test1.txt
xxd -s 4096  -l 8  disk.img            # expect 03
xxd -s 8192  -l 8  disk.img            # expect 03
xxd -s 12416 -l 32 disk.img            # inode 2: type 1, links 1, direct[0]=5
xxd -s 16448 -l 64 disk.img            # entry at 16448, NOT 16384  <- checks A's TODO 4
xxd -s 12288 -l 16 disk.img            # root size now 192 = c0 00 00 00  <- checks A's TODO 11
xxd -s 20480 -l 64 disk.img            # the file text in block 5
md5sum test1.txt                       # must be unchanged
```

### 2. Duplicate rejection — checks A's TODO 3

```bash
./simplefs_adder --input disk.img --file test1.txt
# expect: Error: file already exists in SimpleFS.
```

### 3. Two-block file

```bash
dd if=/dev/zero of=big.dat bs=1 count=5000
rm -f disk.img && ./simplefs_builder --image disk.img
./simplefs_adder --input disk.img --file big.dat
xxd -s 8192  -l 8  disk.img            # expect 07
xxd -s 12416 -l 32 disk.img            # size 5000, direct[0]=5, direct[1]=6, direct[2]=0
```

### 4. Size boundaries

```bash
dd if=/dev/zero of=maxfile.dat bs=1 count=12288
dd if=/dev/zero of=too_big.dat bs=1 count=12289
rm -f disk.img && ./simplefs_builder --image disk.img
./simplefs_adder --input disk.img --file maxfile.dat   # accepted
./simplefs_adder --input disk.img --file too_big.dat   # rejected
```

### 5. Multiple files and first-fit

```bash
rm -f disk.img && ./simplefs_builder --image disk.img
./simplefs_adder --input disk.img --file test1.txt
./simplefs_adder --input disk.img --file test2.txt
./simplefs_adder --input disk.img --file test3.txt
xxd -s 4096 -l 8 disk.img              # expect 0f
xxd -s 8192 -l 8 disk.img              # expect 0f
xxd -s 16384 -l 320 disk.img           # all five entries visible
```

### 6. Failure cases — none may crash

```bash
touch empty.txt
./simplefs_adder --input disk.img --file empty.txt       # inode, no data blocks
./simplefs_adder --input disk.img --file abc123.txt      # source not found
./simplefs_adder --input nothing.img --file test1.txt    # image not found
./simplefs_adder --input disk.img                        # bad arguments
# and a file with a >58 character name
```

### 7. Clean compile

Zero warnings from both files. The four current warnings all point at Member B's empty
TODOs and disappear as they're filled in.

---

## Reference — numbers we keep needing

| Conversion | Formula |
|---|---|
| Block → byte offset | `block * 4096` |
| Inode number → byte offset | `12288 + (inode_no - 1) * 128` |
| Data bitmap index → block | `index + 4` |
| Block → data bitmap index | `block - 4` |

| What | Offset |
|---|---|
| Superblock | 0 |
| Inode bitmap | 4096 |
| Data bitmap | 8192 |
| Inode 1 (root) | 12288 |
| Inode 2 (first file) | 12416 |
| Root directory block | 16384 |
| Entry 0 (`.`) | 16384 |
| Entry 1 (`..`) | 16448 |
| Entry 2 (first file) | 16512 |
| Block 5 (first file's data) | 20480 |

Four numbers that look alike and are not: inode **number** (1–32), inode table **index**
(0–31), data bitmap **index** (0–59), absolute **block** number (4–63).
`direct[]` holds absolute block numbers.

---

## Before submitting

- [ ] Both files compile with zero warnings under `-Wall -Wextra -std=c11`
- [ ] Full test sweep passes
- [ ] Source files unchanged after being added (`md5sum` before and after)
- [ ] Debug `printf` lines removed
- [ ] Submission README written (see below)
- [ ] Zip contains: `simplefs.h`, `simplefs_builder.c`, `simplefs_adder.c`, `README.txt`
- [ ] Zip does **not** contain: executables, `disk.img`, `.o` files, `big.dat` / `maxfile.dat`
- [ ] Submitted through the Google Form — nothing else counts

### Submission README must contain

Group number · both names and student IDs · the two compile commands · execution
examples · brief description of the implementation · **contribution of each member,
listed by TODO number** · known limitations.

---

## Viva prep — 60 of the 100 marks

Both of us get asked about the **whole** project, not just our own half. With only two
members there is nobody to cover a gap.

**Member A should be able to explain B's code:** why the bitmap bit is set inside the
allocation loop, why the copy buffer is re-zeroed each iteration, why `size` is 5000 and
not 8192.

**Member B should be able to explain A's code:** why the magic reads `3153 4653` on disk,
why the root inode has `links = 2`, why data bitmap bit 0 means block 4, why the free-entry
search starts at index 2.

**The exercise that actually works:** each of us reads a raw hex dump of an image with two
files in it and states, from the bytes alone — how many files, their names, their inodes,
their blocks, their sizes.
