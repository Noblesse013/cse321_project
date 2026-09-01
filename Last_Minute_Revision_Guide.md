# SimpleFS — 5-Minute Last-Minute Viva Revision Guide
**CSE321 Lab Term Project**  
*Read this right before walking into your viva room. Everything you need on one page.*

---

## ⚡ 1. The Numbers to Memorize (Instant Recall)

| Constant / Metric | Exact Value | Why / Derivation |
|---|---|---|
| **Total Disk Size** | **262,144 bytes** (256 KiB) | 64 blocks $\times$ 4096 bytes |
| **Block Size** | **4,096 bytes** (4 KiB) | Fixed allocation unit |
| **Total Blocks** | **64 blocks** | Blocks 0–3: Metadata; Blocks 4–63: Data |
| **Total Inodes** | **32 inodes** | 32 inodes $\times$ 128 bytes = 4096 bytes (1 block) |
| **Inode Size** | **128 bytes** | Type (2B) + Links (2B) + Size (4B) + Direct[3] (12B) + Reserved (108B) |
| **Directory Entry Size** | **64 bytes** | Inode_no (4B) + Type (1B) + Name[59] (59B) |
| **Dirents per Block** | **64 entries** | 4096 / 64 = 64 slots in Block 4 |
| **Max Regular Files** | **31 files** | 32 inodes minus Inode 1 (Root) = 31 |
| **Max File Size** | **12,288 bytes** (12 KiB) | 3 direct pointers $\times$ 4096 bytes |
| **Max Filename** | **58 characters** | 59-byte buffer minus 1 byte for `\0` |
| **Magic Number** | **`0x53465331`** | ASCII 'SFS1' (Hex dump shows `31 53 46 53`) |

---

## 🗺️ 2. The 5 Region Offsets (Recite in Order)

$$\text{Byte Offset} = \text{Block Number} \times 4096$$

1. **Block 0 (Offset 0):** Superblock (36B used, records disk layout)
2. **Block 1 (Offset 4096):** Inode Bitmap (Tracks Inodes 1..32)
3. **Block 2 (Offset 8192):** Data Bitmap (Tracks Blocks 4..63)
4. **Block 3 (Offset 12288):** Inode Table (Holds 32 inodes $\times$ 128B)
5. **Blocks 4–63 (Offset 16384+):** Data Region (Block 4 = Root Directory; Block 5 = 1st File)

---

## 📐 3. The 3 Golden Conversion Formulas

1. **Inode Offset on Disk:**  
   $$\text{Offset} = 12288 + (\text{inode\_number} - 1) \times 128$$
   - Inode 1 (Root): `12288`
   - Inode 2 (File 1): `12416`
   - Inode 3 (File 2): `12544`

2. **Data Block $\leftrightarrow$ Bitmap Index:**  
   $$\text{Absolute Block} = \text{Bitmap Index} + 4$$
   $$\text{Bitmap Index} = \text{Absolute Block} - 4$$
   *(Index 0 = Block 4; Index 1 = Block 5)*

3. **Ceiling Division for Blocks:**  
   $$\text{Required Blocks} = \left\lfloor \frac{\text{file\_size} + 4095}{4096} \right\rfloor \quad (\text{Special case: 0 bytes} \to 0\text{ blocks})$$

---

## 🎯 4. The 5 Questions Teachers Love Most (Word-Perfect Answers)

### Q1: Why does `0x53465331` appear as `31 53 46 53` in hex dump?
> **Answer:** "x86 architecture is **Little-Endian**. The least significant byte `0x31` ('1') is stored at the lowest memory address."

### Q2: Why are filenames in directory entries and NOT in inodes?
> **Answer:** "To allow **hard links** (multiple names pointing to the same file) and fast **renaming** without modifying the inode or data blocks."

### Q3: Why does `root_inode` have `links = 2` and `size = 128`?
> **Answer:** "Two entries (`.` and `..`) both point to Inode 1 ($2 \text{ links}$). Each entry is 64 bytes, so $2 \times 64 = 128\text{ bytes}$."

### Q4: Why MUST `set_bit()` be called INSIDE the allocation loop?
> **Answer:** "`find_free_data_block()` uses first-fit for the first clear bit (`0`). If we don't set it immediately, the next iteration will return the **same block number again**, corrupting multi-block files."

### Q5: Why do we zero-fill the buffer with `memset` on each block copy?
> **Answer:** "To ensure that the unused tail of the final data block on disk is clean zero padding (`0x00`) rather than leftover stale memory."

---

## 👥 5. Quick Role Split (Who Did What)

### Member 1 — Fiona (22301024):
- **`simplefs_builder.c` (TODOs 1–6):** Zeroing image, Superblock, Initial Bitmaps, Root Inode, `.` and `..` dirents.
- **`simplefs_adder.c` (TODOs 3, 4, 10, 11):** Duplicate name check (`filename_exists`), searching free directory slot starting from index 2 (`find_free_directory_entry`), writing `dirent_t`, and `root_inode.size += 64`.

### Member 2 — Solaiman (24101031):
- **`simplefs_adder.c` (TODOs 1, 2, 5, 6, 7, 8, 9):** Inode first-fit search, Block first-fit search, `required_blocks` ceiling division, block allocation loop, data copying loop with `memset`, new file inode initialization, and updating inode bitmap.

---

## 💻 6. Essential Terminal Commands

```bash
# Compilation (strict flags)
gcc -Wall -Wextra -std=c11 simplefs_builder.c -o simplefs_builder
gcc -Wall -Wextra -std=c11 simplefs_adder.c   -o simplefs_adder

# Execution
rm -f disk.img
./simplefs_builder --image disk.img
./simplefs_adder --input disk.img --file test1.txt

# Hex Dumps (Inspection)
xxd -l 64 disk.img          # Superblock (Block 0)
xxd -s 4096 -l 16 disk.img  # Inode Bitmap (01 -> 03)
xxd -s 8192 -l 16 disk.img  # Data Bitmap (01 -> 03)
xxd -s 12288 -l 64 disk.img # Root Inode (size 128 -> 192)
xxd -s 16384 -l 192 disk.img# Directory entries (., .., test1.txt)
xxd -s 20480 -l 64 disk.img # First file data (Block 5)
```

---

## 📝 7. Live `nano` Shortcuts

- **Open at line:** `nano +<line_no> <file>` (e.g. `nano +70 simplefs_builder.c`)
- **Go to line inside nano:** `Alt + G` (or `Ctrl + _`)
- **Search text:** `Ctrl + W`
- **Save:** `Ctrl + O` $\to$ Press `Enter`
- **Exit:** `Ctrl + X`
- **Undo / Redo:** `Alt + U` / `Alt + E`

---

## 🚫 8. What NEVER to Say in the Viva

- ❌ *Don't say:* "The inode stores the filename." $\to$ ✅ **"Filenames are stored in directory entries in Block 4."**
- ❌ *Don't say:* "Data bitmap index 0 is Block 0." $\to$ ✅ **"Data bitmap index 0 is Block 4."**
- ❌ *Don't say:* "A 5000-byte file has size 8192." $\to$ ✅ **"`inode.size` is 5000; the extra 3192 bytes are internal fragmentation zero padding."**
- ❌ *Don't say:* "Free directory search starts at 0." $\to$ ✅ **"It starts at index 2 to protect `.` and `..`."**

---

**Take a deep breath. You know every concept, every formula, and every line of code. Good luck! 🚀**
