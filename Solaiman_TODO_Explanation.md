# Solaiman's Complete TODO Explanation & Viva Guide
**CSE321 Lab Term Project — SimpleFS**  
**Group Member:** Md. Solaiman Sache (Student ID: 24101031)  
**Assigned Core Area:** `simplefs_adder.c` (Allocation Engine, Data Copying & Inode Management)

---

## Overview of Solaiman's Assigned Tasks

| File | TODO # | Task Name | Core Purpose |
|---|---|---|---|
| `simplefs_adder.c` | **TODO 1** | `find_free_inode()` | First-fit search in Inode Bitmap (indexes 1..31) $\to$ returns Inode Number (1..32). |
| `simplefs_adder.c` | **TODO 2** | `find_free_data_block()` | First-fit search in Data Bitmap (indexes 0..59) $\to$ returns Absolute Block Number (4..63). |
| `simplefs_adder.c` | **TODO 5** | Calculate `required_blocks` | Ceiling division `(file_size + 4095) / 4096` with special case for 0-byte file. |
| `simplefs_adder.c` | **TODO 6** | Block Allocation Loop | Allocates required blocks and **sets the bitmap bit inside the loop**. |
| `simplefs_adder.c` | **TODO 7** | Data Copying Loop | Copies source file in 4096-byte blocks using `memset` zero-filled buffers. |
| `simplefs_adder.c` | **TODO 8** | New File Inode Init | Sets `type=1`, `links=1`, `size=file_size`, and fills `direct[3]` pointers. |
| `simplefs_adder.c` | **TODO 9** | Mark Inode Bitmap | Marks the new inode allocated using `set_bit(inode_bitmap, free_inode - 1)`. |
| `simplefs_adder.c` | **Extra** | Validation Checks | Rejects filename $> 58$ chars & handles insufficient free block errors. |

---

# PART 1: Deep-Dive into Solaiman's Code (Line-by-Line)

---

### TODO 1: `find_free_inode(unsigned char *bitmap)` — Inode Allocation

#### The Code:
```c
int find_free_inode(unsigned char *bitmap)
{
    /* Search bitmap indexes 1..31 and return INODE NUMBER */
    for (int index = 1; index < TOTAL_INODES; index++) {
        if (!is_bit_set(bitmap, index)) {
            return index + 1; /* Return 1-indexed Inode Number */
        }
    }
    return -1; /* No free inode available */
}
```

#### Detailed Explanation:
1. **Why start searching at `index = 1`?**  
   Index 0 corresponds to **Inode 1**, which is permanently owned by the **Root Directory**. It must never be given to a regular user file.
2. **Why return `index + 1`?**  
   Bitmap indexes are **0-indexed** ($0 \dots 31$), but Inode numbers in SimpleFS are **1-indexed** ($1 \dots 32$).  
   - Index 1 $\implies$ Inode 2
   - Index 2 $\implies$ Inode 3
3. **What is `is_bit_set` doing?**  
   `bitmap[index / 8] & (1u << (index % 8))` tests the exact bit in that byte. If `0` (free), `!is_bit_set` evaluates to true.

#### Viva Question & Answer:
> **Examiner:** *"Why does `find_free_inode` return `index + 1` instead of `index`?"*  
> **Your Answer:** *"Because the caller expects an Inode Number (1 to 32) to store in directory entries and pass to `inode_offset()`. The bitmap itself is 0-indexed (0 to 31), so we add 1 to convert the index to a valid Inode Number."*

---

### TODO 2: `find_free_data_block(unsigned char *bitmap)` — Block Allocation

#### The Code:
```c
int find_free_data_block(unsigned char *bitmap)
{
    /* First-fit search across 60 data blocks; return ABSOLUTE block number */
    for (int index = 0; index < DATA_BLOCKS; index++) {
        if (!is_bit_set(bitmap, index)) {
            return index + DATA_REGION_BLOCK; /* Returns index + 4 */
        }
    }
    return -1; /* Disk data region full */
}
```

#### Detailed Explanation:
1. **Why search indexes `0` to `59` (`DATA_BLOCKS = 60`)?**  
   The disk has 64 total blocks. Blocks 0–3 are reserved for metadata (Superblock, Inode Bitmap, Data Bitmap, Inode Table). The data region contains exactly $64 - 4 = 60$ blocks.
2. **Why return `index + DATA_REGION_BLOCK` (i.e. `index + 4`)?**  
   Data Bitmap index 0 represents **Block 4** on disk. The caller needs an **absolute block number** ($4 \dots 63$) to store in `inode.direct[]` and multiply by 4096 for `fseek()`.

#### Viva Question & Answer:
> **Examiner:** *"Why does Data Bitmap index 0 mean Block 4 and not Block 0?"*  
> **Your Answer:** *"The data bitmap only tracks allocatable user blocks, which start at Block 4. Blocks 0 to 3 are permanent system metadata and can never be allocated to a file, so they are not tracked in the data bitmap."*

---

### TODO 5: Calculate `required_blocks`

#### The Code:
```c
required_blocks = (int)((file_size + BLOCK_SIZE - 1) / BLOCK_SIZE);
if (file_size == 0) {
    required_blocks = 0;
}
```

#### Detailed Explanation:
1. **The Ceiling Division Formula:** `(file_size + 4095) / 4096`
   - $1 \text{ byte} \implies (1 + 4095) / 4096 = 4096 / 4096 = \mathbf{1\text{ block}}$
   - $4096 \text{ bytes} \implies (4096 + 4095) / 4096 = 8191 / 4096 = \mathbf{1\text{ block}}$
   - $4097 \text{ bytes} \implies (4097 + 4095) / 4096 = 8192 / 4096 = \mathbf{2\text{ blocks}}$
   - $5000 \text{ bytes} \implies (5000 + 4095) / 4096 = 9095 / 4096 = \mathbf{2\text{ blocks}}$
   - $12288 \text{ bytes} \implies (12288 + 4095) / 4096 = 16383 / 4096 = \mathbf{3\text{ blocks}}$
2. **The 0-Byte Edge Case:**  
   If `file_size == 0`, `required_blocks` is explicitly set to `0`. A 0-byte file occupies **no data blocks** on disk, but still gets an inode and a directory entry.

---

### TODO 6: Data Block Allocation Loop (The #1 Viva Trap!)

#### The Code:
```c
for (int index = 0; index < required_blocks; index++) {
    int blk = find_free_data_block(data_bitmap);
    if (blk == -1) {
        printf("Error: insufficient free data blocks.\n");
        fclose(source);
        fclose(image);
        return 1;
    }
    allocated_blocks[index] = blk;
    set_bit(data_bitmap, data_bitmap_index(blk)); /* MUST BE INSIDE THE LOOP! */
}
```

#### Detailed Explanation:
1. **Why MUST `set_bit` be inside the loop?**  
   `find_free_data_block()` searches for the first clear bit (`0`). If you don't mark the bit as used immediately, the next iteration will find the **same clear bit** and return the **same block number again**!
   - *Wrong (bit set outside loop):* A 3-block file gets `{5, 5, 5}` $\to$ Block 5 is overwritten 3 times and corrupts the file.
   - *Right (bit set inside loop):* Loop iteration 1 gets Block 5 and sets bit 1 $\to$ iteration 2 sees bit 1 is used and gets Block 6 $\to$ iteration 3 gets Block 7.
2. **What does `data_bitmap_index(blk)` do?**  
   It converts absolute block number back to bitmap index: `blk - 4` (e.g. $5 - 4 = 1$).

---

### TODO 7: Data Copying Loop (Buffer Zeroing)

#### The Code:
```c
for (int index = 0; index < required_blocks; index++) {
    unsigned char buf[BLOCK_SIZE];
    memset(buf, 0, BLOCK_SIZE);                    /* Zero buffer fresh each loop */
    fread(buf, 1, BLOCK_SIZE, source);             /* Read from source */
    fseek(image, (long)allocated_blocks[index] * BLOCK_SIZE, SEEK_SET);
    fwrite(buf, BLOCK_SIZE, 1, image);             /* Write full 4096B block */
}
```

#### Detailed Explanation:
1. **Why `memset(buf, 0, BLOCK_SIZE)` every iteration?**  
   Suppose adding a 5000-byte file:
   - **Iteration 1:** `fread` reads 4096 bytes into `buf` and writes 4096 bytes to Block 5.
   - **Iteration 2:** `fread` reads only the remaining 904 bytes into `buf`.
   - Without `memset`, bytes 904 to 4095 would still contain leftover data from the previous block!
   - With `memset`, bytes 904 to 4095 are clean zeros (`0x00`).
2. **Why always write `BLOCK_SIZE` (4096) bytes with `fwrite`?**  
   File systems operate strictly on whole blocks. Writing the entire zero-padded 4096-byte buffer guarantees the tail of the block on disk is clean zero padding.

---

### TODO 8: New File Inode Initialization

#### The Code:
```c
memset(&new_inode, 0, sizeof(new_inode));
new_inode.type  = TYPE_FILE;           /* 1 = Regular file */
new_inode.links = 1;                   /* 1 directory entry references it */
new_inode.size  = (uint32_t)file_size; /* EXACT source file size (e.g. 5000) */
for (int index = 0; index < required_blocks; index++) {
    new_inode.direct[index] = allocated_blocks[index];
}

fseek(image, inode_offset(free_inode), SEEK_SET);
fwrite(&new_inode, sizeof(new_inode), 1, image);
```

#### Detailed Explanation:
1. **`size` vs. `required_blocks * 4096`:**  
   `new_inode.size` stores the **exact byte size** (e.g. `5000`), NOT the allocated capacity (8192). The remaining 3192 bytes in the block are internal fragmentation padding.
2. **`direct[]` array:**  
   Holds up to 3 absolute block numbers (e.g. `direct[0] = 5`, `direct[1] = 6`, `direct[2] = 0`).
3. **Where is it written?**  
   At byte offset: `12288 + (free_inode - 1) * 128`.

---

### TODO 9: Mark Inode in Bitmap

#### The Code:
```c
set_bit(inode_bitmap, free_inode - 1); /* Note the -1! */
fseek(image, INODE_BITMAP_BLOCK * BLOCK_SIZE, SEEK_SET);
fwrite(inode_bitmap, BLOCK_SIZE, 1, image);
```

#### Detailed Explanation:
- Inode numbers are $1 \dots 32$, but bitmap indexes are $0 \dots 31$.
- To mark Inode 2 as allocated: `set_bit(inode_bitmap, 2 - 1) = set_bit(inode_bitmap, 1)`.
- It is then written back to Block 1 (byte offset `4096`).

---

# PART 2: Solaiman's Live Viva Modification Scenarios

Examiners love asking you to modify your allocation logic live in `nano`:

---

### Scenario 1: *"Change the max file size limit to 2 blocks in `simplefs_adder.c`"*
1. `nano +93 simplefs_adder.c`
2. Change:
   ```c
   if (file_size > 2 * BLOCK_SIZE) {
       printf("Error: file exceeds 2 blocks.\n");
       fclose(source); fclose(image); return 1;
   }
   ```
3. Save (`Ctrl+O` $\to$ `Enter`), Exit (`Ctrl+X`), and recompile:
   ```bash
   gcc -Wall -Wextra -std=c11 simplefs_adder.c -o simplefs_adder
   ```

---

### Scenario 2: *"What happens if you don't clear the buffer with `memset` in TODO 7?"*
- **Viva Answer:** *"The tail of the final data block on disk will contain leftover stale bytes from memory instead of clean zeros. This can be verified with `xxd -s <offset>`."*

---

### Scenario 3: *"Modify `find_free_inode` to search in reverse (from index 31 down to 1)"*
1. `nano +12 simplefs_adder.c`
2. Change loop to:
   ```c
   for (int index = TOTAL_INODES - 1; index >= 1; index--) {
       if (!is_bit_set(bitmap, index)) {
           return index + 1;
       }
   }
   ```
3. First file will now get **Inode 32** instead of Inode 2!

---

# PART 3: Cross-Defense (Understanding Fiona's Part)

> [!IMPORTANT]
> The examiner will test you on Member 1's (Fiona's) contributions. Here is the summary:

1. **`simplefs_builder.c` (Fiona)**:
   - Formats the 256 KiB image with 64 zero-filled blocks.
   - Writes Superblock in Block 0 (`magic = 0x53465331`).
   - Marks Inode 1 and Block 4 used in bitmaps.
   - Creates Root Inode in Block 3 (`type=2`, `links=2`, `size=128`, `direct[0]=4`).
   - Writes `.` and `..` directory entries in Block 4.
2. **Directory Management in `simplefs_adder.c` (Fiona)**:
   - `filename_exists()` (TODO 3): Scans Block 4 for duplicate names before allocations occur.
   - `find_free_directory_entry()` (TODO 4): Scans Block 4 starting from **index 2** (skipping `.` and `..`) for an empty slot (`inode_no == 0`).
   - Writes 64-byte `dirent_t` (TODO 10) and increases `root_inode.size` by `+64` (TODO 11).

---

## Final Checklist for Solaiman

1. **Inode vs Block arithmetic:**  
   - Inode table offset: `12288 + (inode_no - 1) * 128`
   - Absolute block: `bitmap_index + 4`
2. **Bitmap conversions:**  
   - `0x01` $\to$ Bit 0 set (Root)
   - `0x03` $\to$ Bits 0, 1 set (Root + 1st file)
   - `0x07` $\to$ Bits 0, 1, 2 set (Root + 2 files or 2-block file)
3. **Compilation flags:** `gcc -Wall -Wextra -std=c11 simplefs_adder.c -o simplefs_adder`

**All the best, Solaiman! You're ready to ace the viva! 🚀**
