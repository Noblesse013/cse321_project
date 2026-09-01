# Fiona's Complete TODO Explanation & Viva Guide
**CSE321 Lab Term Project — SimpleFS**  
**Group Member:** Mehreen Mallick Fiona (Student ID: 22301024)

---

## Overview of Fiona's Assigned Tasks

| File | TODO # | Task Name | Core Purpose |
|---|---|---|---|
| `simplefs_builder.c` | **TODO 1** | Fill Superblock Fields | Formats Block 0 with metadata describing entire disk geometry. |
| `simplefs_builder.c` | **TODO 2** | Inode Bitmap Root Allocation | Marks Inode 1 as used (bit index 0 in Block 1). |
| `simplefs_builder.c` | **TODO 3** | Data Bitmap Root Allocation | Marks Block 4 as used (bit index 0 in Block 2). |
| `simplefs_builder.c` | **TODO 4** | Root Inode Initialization | Sets root type to directory, links to 2, size to 128, direct[0] to 4. |
| `simplefs_builder.c` | **TODO 5** | `.` Directory Entry | Creates entry pointing to current directory (Inode 1). |
| `simplefs_builder.c` | **TODO 6** | `..` Directory Entry | Creates entry pointing to parent directory (Inode 1). |
| `simplefs_adder.c` | **TODO 3** | `filename_exists()` | Scans Block 4 directory entries to reject duplicate filenames. |
| `simplefs_adder.c` | **TODO 4** | `find_free_directory_entry()` | Searches for an empty slot (`inode_no == 0`), skipping slots 0 and 1. |
| `simplefs_adder.c` | **TODO 10** | Create Directory Entry | Writes new file's name and inode mapping into root directory. |
| `simplefs_adder.c` | **TODO 11** | Update Root Inode Size | Increments root directory size by `+64` bytes (`sizeof(dirent_t)`). |

---

# PART 1: `simplefs_builder.c` (TODOs 1 to 6)

---

### TODO 1: Fill All Superblock Fields

#### The Code:
```c
memset(&sb, 0, sizeof(sb));
sb.magic              = MAGIC_NUMBER;        /* 0x53465331 ("SFS1") */
sb.block_size         = BLOCK_SIZE;          /* 4096 bytes */
sb.total_blocks       = TOTAL_BLOCKS;        /* 64 blocks (256 KiB) */
sb.inode_count        = TOTAL_INODES;        /* 32 inodes */
sb.inode_bitmap_block = INODE_BITMAP_BLOCK;  /* Block 1 (offset 4096) */
sb.data_bitmap_block  = DATA_BITMAP_BLOCK;   /* Block 2 (offset 8192) */
sb.inode_table_block  = INODE_TABLE_BLOCK;   /* Block 3 (offset 12288) */
sb.data_region_block  = DATA_REGION_BLOCK;   /* Block 4 (offset 16384) */
sb.root_inode         = ROOT_INODE;          /* Inode 1 */

fseek(fp, SUPERBLOCK_BLOCK * BLOCK_SIZE, SEEK_SET);
fwrite(&sb, sizeof(sb), 1, fp);
```

#### Detailed Explanation:
1. **What is the Superblock?**  
   It is the master header stored in Block 0 (byte offset `0`). Any program (`simplefs_adder` or inspection tools) reads Block 0 first to learn where every other region is located.
2. **Why use macros instead of raw numbers?**  
   Writing `sb.block_size = BLOCK_SIZE;` is standard C practice. `BLOCK_SIZE` is `#define`d as `4096` in `simplefs.h`. It avoids hardcoded "magic numbers" and ensures single-point configuration.
3. **Why does it take 36 bytes but occupy a full 4096-byte block?**  
   The superblock struct contains 9 `uint32_t` fields ($9 \times 4 = 36\text{ bytes}$). Block 0 is 4096 bytes. Reserving the entire block keeps all subsequent regions aligned to clean 4096-byte block boundaries and leaves room for future expansion.

#### Likely Viva Question & Answer:
> **Examiner:** *"Why is `MAGIC_NUMBER` 0x53465331?"*  
> **Your Answer:** *"In ASCII hex, 0x53='S', 0x46='F', 0x53='S', 0x31='1'. It spells 'SFS1'. It acts as a file signature so our program never accidentally writes into a corrupt or non-SimpleFS file."*

---

### TODO 2: Inode Bitmap — Mark Root Inode Allocated

#### The Code:
```c
set_bit(inode_bitmap, 0);
fseek(fp, INODE_BITMAP_BLOCK * BLOCK_SIZE, SEEK_SET);
fwrite(inode_bitmap, BLOCK_SIZE, 1, fp);
```

#### Detailed Explanation:
1. **The Inode Bitmap (Block 1, offset 4096):**  
   Tracks which of the 32 inodes are free (`0`) or allocated (`1`).
2. **Why index 0?**  
   Inode numbers are 1-indexed ($1 \dots 32$). The root directory is **Inode 1**.  
   $$\text{Bitmap Index} = \text{Inode Number} - 1 = 1 - 1 = 0$$
3. **What does `set_bit(inode_bitmap, 0)` do?**  
   It executes: `inode_bitmap[0 / 8] |= (1u << (0 % 8))` which turns bit 0 of the first byte to `1`. The first byte becomes `0x01` (`00000001` in binary).

---

### TODO 3: Data Bitmap — Mark Root Data Block Allocated

#### The Code:
```c
set_bit(data_bitmap, 0);
fseek(fp, DATA_BITMAP_BLOCK * BLOCK_SIZE, SEEK_SET);
fwrite(data_bitmap, BLOCK_SIZE, 1, fp);
```

#### Detailed Explanation:
1. **The Data Bitmap (Block 2, offset 8192):**  
   Tracks allocation of the 60 data blocks (Blocks 4 through 63).
2. **Why does Data Bitmap Index 0 mean Absolute Block 4?**  
   Blocks 0, 1, 2, and 3 are metadata (Superblock, Inode Bitmap, Data Bitmap, Inode Table). They can never hold file contents. So the data bitmap only starts tracking from Block 4.  
   $$\text{Absolute Block Number} = \text{Data Bitmap Index} + 4$$
   $$\text{Data Bitmap Index} = \text{Absolute Block Number} - 4$$
   The root directory's data entries are stored in **Block 4**, so its bitmap index is $4 - 4 = 0$.

---

### TODO 4: Initialize the Root Inode

#### The Code:
```c
memset(&root_inode, 0, sizeof(root_inode));
root_inode.type      = TYPE_DIRECTORY;       /* 2 = Directory */
root_inode.links     = 2;                    /* '.' and '..' */
root_inode.size      = 2 * sizeof(dirent_t); /* 128 bytes (2 * 64) */
root_inode.direct[0] = ROOT_DATA_BLOCK;      /* Block 4 */

fseek(fp, inode_offset(ROOT_INODE), SEEK_SET);
fwrite(&root_inode, sizeof(root_inode), 1, fp);
```

#### Detailed Explanation:
1. **Where does Inode 1 live?**  
   At the start of the Inode Table (Block 3, byte offset `12288`).  
   $$\text{Offset} = 12288 + (\text{inode\_no} - 1) \times 128 = 12288 + 0 = 12288$$
2. **Why `type = 2`?**  
   `1` is regular file; `2` is directory. Root is a directory.
3. **Why `links = 2`?**  
   Two names point to this inode: `.` and `..`.
4. **Why `size = 128`?**  
   At formatting time, the root directory contains exactly two directory entries (`.` and `..`). Each `dirent_t` is 64 bytes. $2 \times 64 = 128\text{ bytes}$.
5. **Why `direct[0] = 4`?**  
   The actual entries for the root directory are stored in Block 4. `direct[1]` and `direct[2]` remain `0` (unused).

---

### TODO 5 & TODO 6: Initialize `.` and `..` Directory Entries

#### The Code:
```c
/* TODO 5: The '.' entry */
memset(&dot, 0, sizeof(dot));
dot.inode_no = ROOT_INODE;                       /* 1 */
dot.type     = TYPE_DIRECTORY;                  /* 2 */
strncpy(dot.name, ".", sizeof(dot.name) - 1);

/* TODO 6: The '..' entry */
memset(&dotdot, 0, sizeof(dotdot));
dotdot.inode_no = ROOT_INODE;                    /* 1 */
dotdot.type     = TYPE_DIRECTORY;               /* 2 */
strncpy(dotdot.name, "..", sizeof(dotdot.name) - 1);

/* Write to Block 4 (offset 16384) */
fseek(fp, ROOT_DATA_BLOCK * BLOCK_SIZE, SEEK_SET);
fwrite(&dot, sizeof(dot), 1, fp);       /* Written at byte 16384 */
fwrite(&dotdot, sizeof(dotdot), 1, fp); /* Written at byte 16448 */
```

#### Detailed Explanation:
1. **What is a Directory in SimpleFS?**  
   A directory is just a file whose content is an array of `dirent_t` structs (64 bytes each).
2. **Why do both `.` and `..` map to Inode 1?**  
   - `.` means *current directory* $\to$ root (Inode 1).
   - `..` means *parent directory*. Root is the top of the hierarchy and has no parent; by Unix standard convention, the root directory's parent is itself (Inode 1).
3. **Why `sizeof(dot.name) - 1`?**  
   The field `name` has size 59. Using size minus 1 ($59 - 1 = 58$) guarantees room for the string null-terminator `\0`.

---

# PART 2: `simplefs_adder.c` (TODOs 3, 4, 10, 11)

---

### TODO 3: `filename_exists()` — Duplicate Check

#### The Code:
```c
int filename_exists(FILE *image, const char *filename)
{
    dirent_t entry;
    fseek(image, (long)ROOT_DATA_BLOCK * BLOCK_SIZE, SEEK_SET);
    for (int i = 0; i < BLOCK_SIZE / (int)sizeof(dirent_t); i++) {
        if (fread(&entry, sizeof(entry), 1, image) != 1) break;
        if (entry.inode_no == 0) continue;          /* Unused slot */
        if (strcmp(entry.name, filename) == 0) return 1; /* Match found */
    }
    return 0; /* No duplicate */
}
```

#### Detailed Explanation:
1. **Why check for duplicates?**  
   If two files had the same name, path lookup would be ambiguous and corrupt data retrieval.
2. **How does it search?**  
   It seeks to Block 4 (offset `16384`) and iterates through all 64 potential slots ($4096 / 64 = 64$).
3. **Why check `entry.inode_no == 0`?**  
   `inode_no == 0` signifies an empty, unallocated slot. We skip empty slots and only compare names on active entries using `strcmp()`.
4. **Why run this check before allocating blocks?**  
   If we allocated disk blocks first and then discovered a duplicate name, rejecting the file would cause a **space leak** (blocks marked used in the bitmap but belonging to no file).

---

### TODO 4: `find_free_directory_entry()` — Free Slot Lookup

#### The Code:
```c
int find_free_directory_entry(FILE *image)
{
    dirent_t entry;
    /* Skip entries 0 and 1 ('.' and '..') */
    fseek(image, (long)ROOT_DATA_BLOCK * BLOCK_SIZE + 2 * (long)sizeof(dirent_t), SEEK_SET);
    for (int i = 2; i < BLOCK_SIZE / (int)sizeof(dirent_t); i++) {
        if (fread(&entry, sizeof(entry), 1, image) != 1) break;
        if (entry.inode_no == 0) return i;   /* First free slot (index 2..63) */
    }
    return -1; /* Directory full */
}
```

#### Detailed Explanation:
1. **CRITICAL: Why start the loop and seek from index 2?**  
   - Slot 0 (offset `16384`) is `.`
   - Slot 1 (offset `16448`) is `..`  
   If we started searching from index 0, we would overwrite `.` or `..`, destroying the filesystem structure.
2. **How does it know a slot is free?**  
   Because the entire image was zero-filled at formatting time, any unused directory entry has `inode_no == 0`.
3. **What byte offset does the first user file get?**  
   Index 2: $16384 + (2 \times 64) = 16512$.

---

### TODO 10: Create Directory Entry for New File

#### The Code:
```c
memset(&new_entry, 0, sizeof(new_entry));
new_entry.inode_no = free_inode;  /* e.g. Inode 2 */
new_entry.type     = TYPE_FILE;   /* 1 = Regular file */
strncpy(new_entry.name, source_name, sizeof(new_entry.name) - 1);

long pos = ((long)ROOT_DATA_BLOCK * BLOCK_SIZE) + ((long)directory_entry_index * sizeof(dirent_t));
fseek(image, pos, SEEK_SET);
fwrite(&new_entry, sizeof(new_entry), 1, image);
```

#### Detailed Explanation:
1. **What connects the name to the file?**  
   The inode does not store the file's name. This 64-byte `dirent_t` record in Block 4 is the **only place** that maps the human-readable filename (e.g. `"test1.txt"`) to its metadata `free_inode`.
2. **Exact position calculation:**  
   If `directory_entry_index` is `2`, `pos` = $16384 + (2 \times 64) = 16512$. It seeks directly to `16512` and writes the 64 bytes of `new_entry`.

---

### TODO 11: Update Root Inode Size

#### The Code:
```c
fseek(image, inode_offset(ROOT_INODE), SEEK_SET); /* Seek to 12288 */
fread(&root_inode, sizeof(root_inode), 1, image); /* Read current root inode */

root_inode.size += sizeof(dirent_t);              /* Add +64 bytes */

fseek(image, inode_offset(ROOT_INODE), SEEK_SET); /* Seek back to 12288 */
fwrite(&root_inode, sizeof(root_inode), 1, image);/* Write updated root inode */
```

#### Detailed Explanation:
1. **Why does `root_inode.size` increase by 64?**  
   Every new file added takes up one directory slot ($64\text{ bytes}$).  
   - Format: `size = 128` (2 entries: `.` and `..`)
   - Add 1st file: `size = 192` (3 entries)
   - Add 2nd file: `size = 256` (4 entries)
2. **Why seek twice?**  
   `fread()` automatically moves the file position forward by 128 bytes. To overwrite the same root inode on disk, we must call `fseek` again back to offset `12288` before calling `fwrite()`.

---

# PART 3: Viva Whiteboard Quick-Reference

### 1. The 5 Region Offsets:
- **Block 0 (0):** Superblock
- **Block 1 (4096):** Inode Bitmap
- **Block 2 (8192):** Data Bitmap
- **Block 3 (12288):** Inode Table
- **Block 4 (16384):** Data Region (Root Directory)
- **Block 5 (20480):** First User Data Block

### 2. The Conversion Formulas:
- **Inode byte offset:** $12288 + (\text{inode\_number} - 1) \times 128$
- **Data block to bitmap index:** $\text{index} = \text{block\_number} - 4$
- **Bitmap index to data block:** $\text{block\_number} = \text{index} + 4$

### 3. Hex Dump Verification Commands:
```bash
# Check superblock (Block 0):
xxd -l 64 disk.img

# Check inode bitmap (Block 1):
xxd -s 4096 -l 16 disk.img

# Check data bitmap (Block 2):
xxd -s 8192 -l 16 disk.img

# Check root inode (Block 3):
xxd -s 12288 -l 64 disk.img

# Check directory entries (., .., and files in Block 4):
xxd -s 16384 -l 192 disk.img

# Check first file data (Block 5):
xxd -s 20480 -l 64 disk.img
```

---

# PART 4: Compilation & Execution Commands

### 4.1 Strict Compilation (`gcc`)

Always compile both files with strict warning flags:

```bash
gcc -Wall -Wextra -std=c11 simplefs_builder.c -o simplefs_builder
gcc -Wall -Wextra -std=c11 simplefs_adder.c   -o simplefs_adder
```

#### Breakdown of `gcc` Flags:
- `-Wall`: Enables all standard compiler warnings (detects unused variables, bad formats).
- `-Wextra`: Enables extra strict checks (detects signed vs unsigned integer comparisons).
- `-std=c11`: Enforces ISO C11 language standard.
- `-o <filename>`: Sets the output binary name.

---

### 4.2 Step-by-Step Execution Sequence

```bash
# 1. Remove old image so you start from a fresh slate:
rm -f disk.img

# 2. Format a new disk image using simplefs_builder:
./simplefs_builder --image disk.img

# 3. Add test files one by one:
./simplefs_adder --input disk.img --file test1.txt
./simplefs_adder --input disk.img --file test2.txt
./simplefs_adder --input disk.img --file test3.txt
```

---

# PART 5: `nano` Editor Masterclass (For Live Viva Code Modifications)

During the viva, the examiner will frequently ask you to open a file in `nano`, make a small modification, recompile, and inspect what changes on disk.

### 5.1 Essential `nano` Shortcuts Cheat Sheet

| Task | Shortcut in `nano` | How it works |
|---|---|---|
| **Open a file** | `nano <filename>` | e.g. `nano simplefs_builder.c` |
| **Open at specific line** | `nano +<line_no> <file>` | e.g. `nano +70 simplefs_builder.c` (jumps straight to line 70) |
| **Go to line number** | `Alt + G` or `Ctrl + _` | Prompts for line number; type e.g. `72` and press `Enter` |
| **Search text** | `Ctrl + W` | Type word (e.g. `root_inode`) and press `Enter` |
| **Repeat last search** | `Alt + W` | Jumps to next search match |
| **Cut current line** | `Ctrl + K` | Deletes/cuts the line under cursor |
| **Paste / Uncut line** | `Ctrl + U` | Pastes previously cut text at cursor |
| **Save changes (Write Out)** | `Ctrl + O` | Press `Ctrl + O`, then press `Enter` to confirm filename |
| **Exit nano** | `Ctrl + X` | Exits editor (if modified without saving, press `Y` then `Enter`) |
| **Undo last edit** | `Alt + U` | Reverts your previous edit |
| **Redo last edit** | `Alt + E` | Redoes reverted edit |

---

# PART 6: Live Viva Modification Scenarios (Step-by-Step)

Here are the 4 most common live coding requests examiners ask during vivas:

---

### Scenario 1: *"Change the Root Inode Link Count in `simplefs_builder.c`"*

1. **Open the file at line 70:**
   ```bash
   nano +70 simplefs_builder.c
   ```
2. **Locate TODO 4:**
   ```c
   root_inode.links = 2;
   ```
3. **Change it to whatever the examiner requests (e.g. `3`):**
   ```c
   root_inode.links = 3;
   ```
4. **Save and exit `nano`:**
   - Press `Ctrl + O` $\to$ Press `Enter` $\to$ Press `Ctrl + X`
5. **Recompile, rebuild, and inspect:**
   ```bash
   gcc -Wall -Wextra -std=c11 simplefs_builder.c -o simplefs_builder
   rm -f disk.img
   ./simplefs_builder --image disk.img
   xxd -s 12288 -l 16 disk.img
   ```
   *Notice the second 2-byte word changes from `02 00` to `03 00`.*

---

### Scenario 2: *"Modify `simplefs_adder.c` to reject files larger than 2 blocks (8192 bytes)"*

1. **Open `simplefs_adder.c`:**
   ```bash
   nano +93 simplefs_adder.c
   ```
2. **Change the check from `MAX_FILE_SIZE` (12288) to `2 * BLOCK_SIZE`:**
   ```c
   if (file_size > 2 * BLOCK_SIZE) {
       printf("Error: file exceeds 2 blocks.\n");
       fclose(source); fclose(image); return 1;
   }
   ```
3. **Save and exit:**
   - `Ctrl + O` $\to$ `Enter` $\to$ `Ctrl + X`
4. **Recompile and test:**
   ```bash
   gcc -Wall -Wextra -std=c11 simplefs_adder.c -o simplefs_adder
   ```

---

### Scenario 3: *"What happens if `find_free_directory_entry` starts searching from index 0?"*

1. **Open `simplefs_adder.c` at line 56:**
   ```bash
   nano +56 simplefs_adder.c
   ```
2. **If you change `int i = 2` to `int i = 0`:**
   - **Why this breaks the filesystem:** Slot 0 is `.` and Slot 1 is `..`. Although their `inode_no` is currently `1` (non-zero), if a deleted/zeroed entry logic ever checked slot 0 or if slot 0 became 0, the first file would overwrite the `.` or `..` entries.
   - Starting from `i = 2` **guarantees** the directory self-descriptors (`.` and `..`) are permanently protected.

---

### Scenario 4: *"Change the Magic Number to check if adder catches invalid disks"*

1. **Open `simplefs.h`:**
   ```bash
   nano simplefs.h
   ```
2. **Temporarily change `MAGIC_NUMBER`:**
   ```c
   #define MAGIC_NUMBER 0x53465332  /* Changed from SFS1 to SFS2 */
   ```
3. **Recompile `simplefs_adder.c` ONLY (without rebuilding disk):**
   ```bash
   gcc -Wall -Wextra -std=c11 simplefs_adder.c -o simplefs_adder
   ./simplefs_adder --input disk.img --file test1.txt
   ```
4. **Expected Output:**
   ```
   Error: invalid SimpleFS image.
   ```
   *This proves your magic number validation works properly!*

---

# PART 7: Whole-Project Technical Mastery (Cross-Contribution Defense)

> [!IMPORTANT]
> **Official Viva Guideline:**  
> *"The viva will consist of technical questions from the project. Each of the members is expected to have an idea of the complete project as the questions will not be limited to your specific contributions only."*

To score full marks, you must understand your partner's (Member 2 - Md. Solaiman Sache) assigned tasks and how the entire system functions as one cohesive unit.

---

### 7.1 Partner's Specific Tasks (Member 2 — Solaiman's Code)

```
+---------------------------------------------------------------------------------------+
| MEMBER 2'S TASKS (simplefs_adder.c)                                                   |
+---------------------------------------------------------------------------------------+
| TODO 1: find_free_inode()          - First-fit search in Inode Bitmap (index 1..31)   |
| TODO 2: find_free_data_block()     - First-fit search in Data Bitmap (index 0..59)    |
| TODO 5: Calculate required_blocks  - Ceiling division formula; 0-byte file = 0 blocks |
| TODO 6: Data Block Allocation Loop - Allocate blocks and set bitmap bits inside loop   |
| TODO 7: Data Copying Loop          - Copy data in 4096B blocks with memset zeroing    |
| TODO 8: New File Inode Init        - Fill type=1, links=1, size, direct[3] pointers    |
| TODO 9: Inode Bitmap Update        - Mark free_inode as used (set_bit index - 1)      |
+---------------------------------------------------------------------------------------+
```

---

### 7.2 Deep-Dive into Member 2's Code (Be Ready to Explain These!)

#### 1. `find_free_inode(unsigned char *bitmap)` (TODO 1)
```c
for (int index = 1; index < TOTAL_INODES; index++) {
    if (!is_bit_set(bitmap, index)) {
        return index + 1; /* Return Inode NUMBER (1-indexed) */
    }
}
return -1;
```
* **Why start at `index = 1`?** Index 0 is Inode 1 (Root Directory), which is permanently allocated.
* **Why return `index + 1`?** The function returns an Inode **Number** ($1 \dots 32$). The caller passes this number to `inode_offset()`.

---

#### 2. `find_free_data_block(unsigned char *bitmap)` (TODO 2)
```c
for (int index = 0; index < DATA_BLOCKS; index++) {
    if (!is_bit_set(bitmap, index)) {
        return index + DATA_REGION_BLOCK; /* Return ABSOLUTE block number (index + 4) */
    }
}
return -1;
```
* **Why return `index + 4`?** Because the first data block on disk is Block 4. The returned value is an **absolute block number** stored directly into `inode.direct[]`.

---

#### 3. Calculating `required_blocks` (TODO 5)
```c
required_blocks = (int)((file_size + BLOCK_SIZE - 1) / BLOCK_SIZE);
if (file_size == 0) {
    required_blocks = 0;
}
```
* **Ceiling Division:** `(file_size + 4095) / 4096` ensures:
  - 1 byte to 4096 bytes $\to 1$ block.
  - 4097 bytes to 8192 bytes $\to 2$ blocks.
  - 8193 bytes to 12288 bytes $\to 3$ blocks.
* **0-Byte File:** Uses **0 data blocks**, but still gets an inode (`size = 0`, `direct = {0,0,0}`) and a directory entry.

---

#### 4. The Allocation Loop & Crucial Trap (TODO 6)
```c
for (int index = 0; index < required_blocks; index++) {
    int blk = find_free_data_block(data_bitmap);
    if (blk == -1) { /* error: insufficient blocks */ }
    allocated_blocks[index] = blk;
    set_bit(data_bitmap, data_bitmap_index(blk)); /* MUST BE SET INSIDE THE LOOP! */
}
```
* **The Trap Question:** *"Why must `set_bit` be called inside the loop?"*  
  **Your Answer:** *"Because `find_free_data_block` uses first-fit to find the first clear bit (`0`). If we don't set the bit immediately, the next iteration will find the same `0` bit and return the exact same block number again!"*

---

#### 5. Data Copying with `memset` Buffer (TODO 7)
```c
for (int index = 0; index < required_blocks; index++) {
    unsigned char buf[BLOCK_SIZE];
    memset(buf, 0, BLOCK_SIZE); /* Zero-fill buffer each iteration */
    fread(buf, 1, BLOCK_SIZE, source);
    fseek(image, (long)allocated_blocks[index] * BLOCK_SIZE, SEEK_SET);
    fwrite(buf, BLOCK_SIZE, 1, image); /* Always write full 4096 bytes */
}
```
* **The Trap Question:** *"If a file is 5,000 bytes, why do we write 4096 bytes on the second block when only 904 bytes were read?"*  
  **Your Answer:** *"The buffer is zeroed with `memset` before reading. For block 2, `fread` reads 904 bytes and the remaining 3192 bytes stay `0`. Writing the full 4096 bytes guarantees the tail of the block on disk is clean zero padding rather than stale garbage data."*

---

#### 6. Initializing the New File Inode (TODO 8 & 9)
```c
new_inode.type  = TYPE_FILE;         /* 1 */
new_inode.links = 1;                 /* 1 link */
new_inode.size  = (uint32_t)file_size; /* Actual file size (e.g. 5000), NOT 8192 */
for (int index = 0; index < required_blocks; index++) {
    new_inode.direct[index] = allocated_blocks[index];
}

/* Write inode to inode table */
fseek(image, inode_offset(free_inode), SEEK_SET);
fwrite(&new_inode, sizeof(new_inode), 1, image);

/* Mark inode bitmap */
set_bit(inode_bitmap, free_inode - 1); /* Note the -1: Inode 2 is bit index 1 */
```

---

### 7.3 End-to-End Execution Flow (The 16-Step Pipeline)

Be ready to explain the complete lifetime of adding a file to SimpleFS:

1. Open `disk.img` in `"rb+"` mode (preserve contents).
2. Read Superblock from Block 0; verify `magic == 0x53465331`.
3. Open source file in `"rb"` mode.
4. Measure source file size using `fseek(SEEK_END)` and `ftell()`.
5. Reject if `file_size > 12288` bytes (exceeds 3 direct pointers).
6. Reject if filename length $> 58$ characters.
7. Calculate `required_blocks = (file_size + 4095) / 4096`.
8. Scan root directory (Block 4); reject duplicate filename (**Fiona - TODO 3**).
9. Read Inode Bitmap (Block 1); find first free inode (**Solaiman - TODO 1**).
10. Read Data Bitmap (Block 2); allocate blocks and mark bits (**Solaiman - TODO 6**).
11. Copy file data from source to allocated blocks (**Solaiman - TODO 7**).
12. Write new file's `inode_t` to Inode Table at Block 3 (**Solaiman - TODO 8**).
13. Write updated Inode Bitmap and Data Bitmap back to disk.
14. Find free directory slot starting at index 2 (**Fiona - TODO 4**).
15. Write `dirent_t` into Block 4 (**Fiona - TODO 10**).
16. Increase `root_inode.size` by `+64` and write back to disk (**Fiona - TODO 11**).

---

### 7.4 Summary of Corner Cases & Edge Conditions

| Scenario | Behavior / Handling in Code |
|---|---|
| **0-byte file** | Needs **0 data blocks**. Receives 1 inode (`size = 0`, `direct = {0,0,0}`) and 1 directory entry. |
| **4096-byte file** | Needs exactly **1 data block**. `(4096 + 4095) / 4096 = 1`. |
| **5000-byte file** | Needs **2 data blocks**. Inode stores `size = 5000`. Block 2 has 904 real bytes + 3192 zero padding. |
| **12288-byte file** | Accepted (fills all 3 direct pointers). |
| **12289-byte file** | **Rejected** with error message (`> MAX_FILE_SIZE`). |
| **58-character name** | **Accepted** (fills 58 chars + 1 null terminator = 59-byte buffer). |
| **59-character name** | **Rejected** (no room for null terminator). |
| **Duplicate file** | **Rejected** before any bitmap allocation to prevent space leaks. |

---

## Final Words for the Viva

You have mastered:
- Every line and syntax detail of your assigned code (`simplefs_builder.c` and `simplefs_adder.c`).
- How to compile with `gcc`, edit live in `nano`, and inspect bytes with `xxd`.
- The complete end-to-end architecture and your teammate's allocation mechanisms.

**All the best for your viva! You are fully prepared to get top marks! 🚀**
