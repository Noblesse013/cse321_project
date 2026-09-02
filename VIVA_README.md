# SimpleFS — Ultimate Viva Guide & Technical Reference Manual
**Course:** CSE321: Operating Systems Lab Term Project  
**System:** SimpleFS (Simple File System in C)  
**Disk Image:** `disk.img` (256 KiB, 64 Blocks $\times$ 4096 Bytes)

---

## 📑 Table of Contents
1. [Disk Block Map & "In Which Block is What Generated?"](#1-disk-block-map--in-which-block-is-what-generated)
2. ["What Happens If This `if/else` or TODO Line is Removed?" (Code Mutation Analysis)](#2-what-happens-if-this-ifelse-or-todo-line-is-removed-code-mutation-analysis)
3. [Bit Manipulation, Masking & "How Bits are Inverted"](#3-bit-manipulation-masking--how-bits-are-inverted)
4. [The 16 Canonical Test Cases & Inspection Commands](#4-the-16-canonical-test-cases--inspection-commands)
5. [The Whiteboard Arithmetic & Number Conversions](#5-the-whiteboard-arithmetic--number-conversions)
6. [Quick Viva Survival Cheat Sheet](#6-quick-viva-survival-cheat-sheet)

---

## 1. Disk Block Map & "In Which Block is What Generated?"

Teachers frequently ask: *"If I open `disk.img`, what is stored in each block, at what byte offset, and why?"*

### 1.1 Master Disk Layout Table

$$\text{Byte Offset} = \text{Block Number} \times 4096$$

| Block # | Byte Offset | Region Name | Contents / Structures Written | Hex Dump Command |
|---|---|---|---|---|
| **Block 0** | `0` | **Superblock** | 36-byte `superblock_t` (magic `0x53465331`, 4096, 64, 32, pointers to blocks 1, 2, 3, 4). Bytes 36–4095 are zero-padded. | `xxd -l 128 disk.img` |
| **Block 1** | `4096` | **Inode Bitmap** | Tracks 32 inodes. Byte 0 is `0x01` initially (Inode 1 allocated for Root). After 1 file added, it becomes `0x03`. | `xxd -s 4096 -l 16 disk.img` |
| **Block 2** | `8192` | **Data Bitmap** | Tracks 60 data blocks (Blocks 4–63). Byte 0 is `0x01` initially (Block 4 allocated for Root Directory). After adding 1 file, it becomes `0x03`. | `xxd -s 8192 -l 16 disk.img` |
| **Block 3** | `12288` | **Inode Table** | 32 inodes $\times$ 128 bytes = 4096 bytes (fits in exactly 1 block).<br>• Inode 1 (Root): `12288`<br>• Inode 2 (File 1): `12416`<br>• Inode 3 (File 2): `12544` | `xxd -s 12288 -l 256 disk.img` |
| **Block 4** | `16384` | **Root Directory** | Holds 64 directory entries (`dirent_t`, 64 bytes each).<br>• Slot 0 (`.`): points to Inode 1<br>• Slot 1 (`..`): points to Inode 1<br>• Slot 2 (File 1): points to Inode 2 | `xxd -s 16384 -l 192 disk.img` |
| **Block 5** | `20480` | **First File Data** | Actual content of the first added file (`test1.txt`). Unused bytes are zero-padded. | `xxd -s 20480 -l 64 disk.img` |
| **Blocks 6–63** | `24576`–`258048` | **Remaining Data** | Additional data blocks for multi-block files or subsequent files. | `xxd -s 24576 -l 64 disk.img` |

---

### 1.2 Step-by-Step Block Generation

#### A. When `simplefs_builder` creates `disk.img`:
1. **Initializes the raw file:** Writes 64 zeroed blocks ($64 \times 4096 = 262,144$ bytes).
2. **Block 0 (Offset 0):** Writes the 36-byte superblock (`sb.magic = 0x53465331`, `sb.block_size = 4096`, `sb.total_blocks = 64`, `sb.inode_count = 32`, `sb.root_inode = 1`).
3. **Block 1 (Offset 4096):** Sets bit 0 (`set_bit(inode_bitmap, 0)`) to permanently mark Inode 1 as allocated for the root directory.
4. **Block 2 (Offset 8192):** Sets bit 0 (`set_bit(data_bitmap, 0)`) to permanently mark Block 4 as allocated for the root directory data.
5. **Block 3 (Offset 12288):** Writes `root_inode` (`type = 2`, `links = 2`, `size = 128`, `direct[0] = 4`).
6. **Block 4 (Offset 16384):** Writes two 64-byte directory entries:
   - Slot 0: `.` $\to$ `inode_no = 1`, `type = 2`, `name = "."`
   - Slot 1: `..` $\to$ `inode_no = 1`, `type = 2`, `name = ".."`

#### B. When `simplefs_adder` adds a file (e.g., `test1.txt`):
1. **Block 5 (Data Region):** Copies the file contents into Block 5 using a `memset` zeroed buffer.
2. **Block 3 (Inode Table):** Writes `new_inode` at offset $12288 + (\text{inode} - 1) \times 128 = 12416$, and updates `root_inode.size` (+64 bytes).
3. **Block 1 (Inode Bitmap):** Flips bit 1 from `0` to `1` (byte 0 transitions from `0x01` $\to$ `0x03`).
4. **Block 2 (Data Bitmap):** Flips bit 1 from `0` to `1` (byte 0 transitions from `0x01` $\to$ `0x03`).
5. **Block 4 (Root Directory):** Writes `new_entry` (`inode_no = 2`, `type = 1`, `name = "test1.txt"`) in free slot 2.

---

## 2. "What Happens If This `if/else` or TODO Line is Removed?" (Code Mutation Analysis)

Teachers love to point at specific lines in `simplefs_builder.c` and `simplefs_adder.c` and ask: *"What breaks if I delete or comment out this line?"*

### Case 1: Removing `set_bit(data_bitmap, data_bitmap_index(blk))` inside the allocation loop (TODO 6)
```c
for (int index = 0; index < required_blocks; index++) {
    int blk = find_free_data_block(data_bitmap);
    allocated_blocks[index] = blk;
    set_bit(data_bitmap, data_bitmap_index(blk)); // <-- IF REMOVED
}
```
* **Fatal Consequence:** **Catastrophic data corruption for multi-block files.**
* **Explanation:** `find_free_data_block()` performs a first-fit search for the first clear bit (`0`). If you do not mark the allocated bit immediately in memory, the next loop iteration will see the exact same bit as free and return the **same block number again** (`direct[0] = 5`, `direct[1] = 5`). Block 5 will be overwritten by block 2's data, and half the file is permanently destroyed.

---

### Case 2: Removing `if (file_size == 0) required_blocks = 0;` (TODO 5)
```c
required_blocks = (int)((file_size + BLOCK_SIZE - 1) / BLOCK_SIZE);
if (file_size == 0) { // <-- IF REMOVED
    required_blocks = 0;
}
```
* **Consequence:** In integer math $(0 + 4095)/4096 = 0$. However, if an alternative ceiling formula like `(file_size / BLOCK_SIZE) + 1` was used, it would erroneously allocate 1 data block (4096 bytes) for an empty file.
* **Why this check is mandatory:** An empty 0-byte file must consume an inode, but **0 data blocks** (`direct[0..2] = 0`).

---

### Case 3: Removing `if (sb.magic != MAGIC_NUMBER)` (Superblock Verification)
```c
if (sb.magic != MAGIC_NUMBER) { // <-- IF REMOVED
    printf("Error: invalid SimpleFS image.\n");
    fclose(image); return 1;
}
```
* **Consequence:** **Arbitrary file corruption.**
* **Explanation:** If a user accidentally passes a non-SimpleFS file (e.g. an MP3, JPEG, or OS executable) to `--input`, the program would treat it as a valid disk. The adder would seek to hardcoded offsets (`4096`, `8192`, `12288`, `16384`) and overwrite random bytes, permanently destroying the file.

---

### Case 4: Removing `if (filename_exists(image, source_name))` (Duplicate Check)
```c
if (filename_exists(image, source_name)) { // <-- IF REMOVED
    printf("Error: file already exists in SimpleFS.\n");
    fclose(source); fclose(image); return 1;
}
```
* **Consequence:** Duplicate entries with the identical filename will be added to the directory. This violates the fundamental POSIX requirement of unique filenames within a single directory, creating ambiguity over which inode is referenced.

---

### Case 5: Removing `root_inode.size += sizeof(dirent_t);` (TODO 11)
```c
root_inode.size += sizeof(dirent_t); // <-- IF REMOVED
```
* **Consequence:** The root directory's size will remain permanently frozen at 128 bytes ($2 \times 64$, for `.` and `..`). Any file-listing utility (`ls` equivalent) calculating directory entry count from `root_inode.size / 64` will only see the two dot entries, making added files invisible.

---

### Case 6: Changing `find_free_directory_entry` to start from `i = 0` instead of `i = 2`
```c
for (int i = 2; i < BLOCK_SIZE / (int)sizeof(dirent_t); i++) // <-- IF i STARTS AT 0
```
* **Consequence:** Slots 0 and 1 are permanently reserved for `.` and `..`. If a corruption causes slot 0 or 1 to show `inode_no == 0`, or if an uninitialized entry is encountered, user files will overwrite `.` or `..`, destroying directory navigation and parent linking.

---

### Case 7: Changing `find_free_inode` to start at `index = 0` instead of `index = 1`
```c
for (int index = 1; index < TOTAL_INODES; index++) // <-- IF index STARTS AT 0
```
* **Consequence:** Bitmap index 0 corresponds to Inode 1 (Root Inode). If index 0 were returned as free, a regular file would overwrite Inode 1 in the Inode Table. The root directory would become a regular file, corrupting the entire file system.

---

### Case 8: Removing `memset(buf, 0, BLOCK_SIZE)` before `fread` in TODO 7
```c
unsigned char buf[BLOCK_SIZE];
memset(buf, 0, BLOCK_SIZE); // <-- IF REMOVED
fread(buf, 1, BLOCK_SIZE, source);
```
* **Consequence:** **Information leakage / dirty data on disk.**
* If `source` is 50 bytes, `fread` only fills 50 bytes. The remaining 4046 bytes in `buf` would contain uninitialized stack memory/garbage, which gets written directly into the disk image instead of clean zeros.

---

### Case 9: Removing `set_bit(inode_bitmap, 0)` & `set_bit(data_bitmap, 0)` in `simplefs_builder.c`
```c
set_bit(inode_bitmap, 0); // <-- IF REMOVED
set_bit(data_bitmap, 0);  // <-- IF REMOVED
```
* **Consequence:** Inode 1 and Block 4 will appear as free (`0`). The very first run of `simplefs_adder` will allocate Inode 1 and Block 4 to a user file, wiping out the root directory and root inode entirely.

---

## 3. Bit Manipulation, Masking & "How Bits are Inverted"

Teachers frequently ask students to write bitwise operations on the whiteboard.

### 3.1 Setting a Bit (`set_bit`) — Marking Allocated
```c
void set_bit(unsigned char *bitmap, int index) {
    bitmap[index / 8] |= (1u << (index % 8));
}
```
* `index / 8`: Finds the target byte in the bitmap (e.g., bit 17 is in byte $17 / 8 = 2$).
* `index % 8`: Finds the bit position within that byte ($17 \pmod 8 = 1$).
* `1u << (index % 8)`: Generates a bitmask with a single `1` at the target position (e.g., `00000010`).
* `|=` (Bitwise OR): Sets the target bit to `1` without changing any surrounding bits.

---

### 3.2 Checking a Bit (`is_bit_set`) — Testing If Allocated
```c
int is_bit_set(unsigned char *bitmap, int index) {
    return bitmap[index / 8] & (1u << (index % 8));
}
```
* `&` (Bitwise AND): Isolates the bit. Returns non-zero if the bit is `1` (allocated), and `0` if the bit is `0` (free).

---

### 3.3 How Bits are Inverted / Cleared (`clear_bit`) — Freeing a Resource
Teachers often ask: *"How would you delete a file or invert/clear a bit?"*

```c
void clear_bit(unsigned char *bitmap, int index) {
    bitmap[index / 8] &= ~(1u << (index % 8));
}
```
* **The Bit Inversion Operator (`~`):**
  - Suppose we want to clear bit 2: `(1u << 2)` produces: `00000100` (binary).
  - Applying bitwise NOT `~` **inverts every bit**: `11111011` (all 1s, except a 0 at target bit 2).
  - Applying bitwise AND `&=` with this inverted mask forces the target bit to `0` while leaving every other bit untouched!

* **Flipping / Toggling a Bit (Bitwise XOR `^`):**
```c
bitmap[index / 8] ^= (1u << (index % 8));
```
  - `0 ^ 1 = 1` and `1 ^ 1 = 0` (unconditionally flips the bit).

---

### 3.4 Why Hex Bitmap Values Progress as `0x01` $\to$ `0x03` $\to$ `0x07` $\to$ `0x0F`
In x86 (little-endian), bit 0 is the Least Significant Bit (LSB):

| Bit Pattern | Binary Value | Hex Dump Output | SimpleFS Meaning |
|---|---|---|---|
| Bit 0 only | `00000001` | `0x01` | Initial state (Root Inode / Root Block 4) |
| Bits 0, 1 | `00000011` | `0x03` | Root + 1 single-block file added |
| Bits 0, 1, 2 | `00000111` | `0x07` | Root + 2-block file added (Blocks 4, 5, 6 used) |
| Bits 0, 1, 2, 3 | `00001111` | `0x0F` | Root + 3-block file added (Blocks 4, 5, 6, 7 used) |

---

## 4. The 16 Canonical Test Cases & Inspection Commands

Every viva test scenario directly maps to the 16 test cases from Section 19 of the project specifications and edge-case testing.

| # | Test Scenario | Execution Command | Inspection Command | Expected Result |
|---|---|---|---|---|
| **1** | **Empty File System** | `./simplefs_builder --image disk.img` | `ls -l disk.img` | Exactly **262,144 bytes**. |
| **2** | **Superblock Integrity** | Fresh image | `xxd -l 128 disk.img` | Block 0 starts with `31 53 46 53` (`SFS1`), followed by `00 10 00 00` (4096), `40 00 00 00` (64). |
| **3** | **Initial Inode Bitmap** | Fresh image | `xxd -s 4096 -l 16 disk.img` | First byte is `01` (Inode 1 allocated for Root). |
| **4** | **Initial Data Bitmap** | Fresh image | `xxd -s 8192 -l 16 disk.img` | First byte is `01` (Block 4 allocated for Root Directory). |
| **5** | **Root Directory Entries** | Fresh image | `xxd -s 16384 -l 128 disk.img` | ASCII column shows `.` (slot 0) and `..` (slot 1), both with `inode_no = 1`. |
| **6** | **Add Small File** | `./simplefs_adder --input disk.img --file test1.txt` | Terminal output | Prints `test1.txt added successfully to disk.img`. |
| **7** | **Inode Allocation (1 File)** | After adding `test1.txt` | `xxd -s 4096 -l 8 disk.img` | Inode bitmap first byte transitions from `01` $\to$ `03` (Inode 2 allocated). |
| **8** | **Data Allocation (1 File)** | After adding `test1.txt` | `xxd -s 8192 -l 8 disk.img` | Data bitmap first byte transitions from `01` $\to$ `03` (Block 5 allocated). |
| **9** | **File Content Verification** | After adding `test1.txt` | `xxd -s 20480 -l 64 disk.img` | Block 5 (offset `20480`) matches the exact ASCII content of `test1.txt`. |
| **10** | **Two-Block File (5000B)** | `test2.txt` (5000 bytes) added | `xxd -s 8192 -l 8 disk.img` | Data bitmap becomes `07` (`00000111` $\to$ Blocks 4, 5, 6 allocated). `direct[0]=5`, `direct[1]=6`. |
| **11** | **Max File Size Boundary** | Add 12,288-byte vs 12,289-byte file | Execution output | 12,288-byte file is **accepted** (uses 3 blocks); 12,289-byte file is **rejected**: `"Error: file is too large for SimpleFS."` |
| **12** | **Multiple Files (First-Fit)** | Add `test1.txt`, `test2.txt`, `test3.txt` | `xxd -s 16384 -l 256 disk.img` | Inodes 2, 3, 4 assigned; data placed in Blocks 5, 6, 7 under first-fit. |
| **13** | **Duplicate Filename Check** | Re-run adder with same filename | Execution output | Rejected with: `"Error: file already exists in SimpleFS."` |
| **14** | **Missing Source File** | `./simplefs_adder --input disk.img --file ghost.txt` | Execution output | Rejected safely with: `"Error: source file not found."` (No crash/segfault). |
| **15** | **Missing Disk Image** | `./simplefs_adder --input fake.img --file test1.txt` | Execution output | Rejected safely with: `"Error: file-system image not found."` (No crash/segfault). |
| **16** | **Zero-Byte File Handling** | `touch empty.txt` & add to image | `xxd -s 12288 -l 256 disk.img` | Inode is allocated, `size = 0`, but `direct[0..2] = 0` (**0 data blocks consumed**). |

---

## 5. The Whiteboard Arithmetic & Number Conversions

### Conversion 1: Inode Disk Offset Formula
$$\text{Disk Offset} = (\text{INODE\_TABLE\_BLOCK} \times \text{BLOCK\_SIZE}) + ((\text{inode\_number} - 1) \times \text{sizeof(inode\_t)})$$
$$\text{Disk Offset} = 12288 + (\text{inode\_number} - 1) \times 128$$
* Inode 1 (Root): $12288 + 0 = 12288$
* Inode 2 (File 1): $12288 + 128 = 12416$
* Inode 3 (File 2): $12288 + 256 = 12544$

---

### Conversion 2: Data Block $\leftrightarrow$ Bitmap Index
$$\text{Absolute Block Number} = \text{Data Bitmap Index} + 4$$
$$\text{Data Bitmap Index} = \text{Absolute Block Number} - 4$$
* Bitmap Index `0` $\iff$ Block `4` (Root Directory)
* Bitmap Index `1` $\iff$ Block `5` (First file data)
* Bitmap Index `2` $\iff$ Block `6` (Second data block)

---

### Conversion 3: Ceiling Division for Block Count
$$\text{Required Blocks} = \left\lfloor \frac{\text{file\_size} + 4095}{4096} \right\rfloor \quad (\text{Special Case: } \text{file\_size} = 0 \implies 0\text{ blocks})$$

---

## 6. Quick Viva Survival Cheat Sheet

1. **Where is filename stored?** **Never in the inode.** Filenames are stored exclusively in directory entries inside Block 4 (`dirent_t`).
2. **Why is `0x53465331` displayed as `31 53 46 53`?** x86 is **Little-Endian**; least significant byte `0x31` ('1') is stored at the lowest address.
3. **Why does root inode have `links = 2`?** Because two directory entries (`.` and `..`) both point to Inode 1.
4. **Why `root_inode.size = 128`?** 2 directory entries $\times$ 64 bytes each = 128 bytes.
5. **How to invert a bit?** Use bitwise NOT `~` with bitwise AND: `bitmap[byte] &= ~(1u << bit)`.
