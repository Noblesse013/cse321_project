# SimpleFS — Basic Viva Questions & Word-Perfect Answers
**CSE321 Operating Systems Lab Term Project**  
*A comprehensive guide covering all fundamental viva questions teachers ask to assess student understanding.*

---

# TIER 1: General OS & File System Fundamentals (Warm-Up Questions)

### Q1: What is a file system?
> **Answer:** "A file system is an agreement and set of data structures that organizes raw bytes on a storage device into files, directories, metadata, and free space management."

---

### Q2: What is an Inode (Index Node)?
> **Answer:** "An inode is a fixed-size metadata data structure that stores everything about a file — such as its type, link count, byte size, and data block pointers — **except its name**."

---

### Q3: Why is the filename NOT stored inside the Inode?
> **Answer:** "For two main reasons:
> 1. **Hard Links:** It allows multiple directory entries (different names/paths) to point to the exact same inode.
> 2. **Efficient Renaming:** Renaming a file only updates a 64-byte directory entry; the inode and underlying data blocks remain completely untouched."

---

### Q4: What is a Superblock?
> **Answer:** "The superblock is the master metadata block located at Block 0 that records the entire disk geometry — including block size, total block count, inode count, and the starting block numbers of all other regions."

---

### Q5: What is a Bitmap? Why use it instead of a linked list?
> **Answer:** "A bitmap uses 1 bit per resource (0 = free, 1 = allocated). We use it because it is extremely compact (32 inodes need only 4 bytes; 60 blocks need only 8 bytes) and allows fast $O(1)$ bitwise checking."

---

### Q6: What is a Directory in Unix and SimpleFS?
> **Answer:** "A directory is an ordinary file whose contents happen to be a table of name-to-inode mappings (`dirent_t` records)."

---

### Q7: What are `.` and `..`?
> **Answer:** "`.` represents the current directory, and `..` represents the parent directory. In the root directory, since there is no parent above it, both `.` and `..` point to Inode 1 (itself)."

---

### Q8: What is Internal Fragmentation? Where does it occur here?
> **Answer:** "Internal fragmentation is wasted space inside an allocated block when a file does not fill the full block. For example, a 1-byte file still consumes a full 4096-byte block, wasting 4095 bytes."

---

### Q9: What is External Fragmentation? Does SimpleFS have it?
> **Answer:** "External fragmentation occurs when free memory is broken into unusable small pieces. SimpleFS does **not** have external fragmentation because all blocks are fixed-size (4096 bytes) and interchangeable."

---

### Q10: What are Direct Pointers vs. Indirect Pointers?
> **Answer:** "Direct pointers store the absolute block numbers of actual file data (SimpleFS has 3 direct pointers = 12 KiB max). Indirect pointers point to a block that contains a list of more block pointers, enabling larger file sizes."

---

# TIER 2: SimpleFS Architecture & Numbers (Whiteboard Math)

### Q11: What is the total image size, and why?
> **Answer:** "Exactly **262,144 bytes (256 KiB)**, because SimpleFS has **64 blocks** of **4096 bytes** each ($64 \times 4096 = 262144$)."

---

### Q12: Name the 5 regions of SimpleFS and their starting byte offsets.
> **Answer:**  
> 1. **Block 0 (offset 0):** Superblock  
> 2. **Block 1 (offset 4096):** Inode Bitmap  
> 3. **Block 2 (offset 8192):** Data Bitmap  
> 4. **Block 3 (offset 12288):** Inode Table  
> 5. **Blocks 4–63 (offset 16384+):** Data Region (Block 4 is the Root Directory)

---

### Q13: How many inodes fit in the Inode Table?
> **Answer:** "Exactly **32 inodes**. Each inode is **128 bytes**, so $32 \times 128 = 4096\text{ bytes}$, fitting perfectly into exactly 1 block (Block 3)."

---

### Q14: How many directory entries fit in Block 4?
> **Answer:** "Exactly **64 entries**. Each `dirent_t` is **64 bytes**, so $4096 / 64 = 64$ entries."

---

### Q15: What is the maximum number of regular files SimpleFS can hold?
> **Answer:** "**31 regular files**. Although there are 64 directory slots, there are only 32 inodes, and Inode 1 is permanently reserved for the root directory ($32 - 1 = 31$)."

---

### Q16: Why is the maximum filename length 58 characters?
> **Answer:** "The `name` buffer is 59 bytes. In C, strings must end with a null terminator (`\0`). Thus, 58 characters + 1 null terminator = 59 bytes."

---

### Q17: What is the maximum file size supported? Why?
> **Answer:** "**12,288 bytes (12 KiB)**, because each inode contains exactly **3 direct block pointers**, and $3 \times 4096 = 12288\text{ bytes}$."

---

# TIER 3: C Programming & Low-Level Systems Concepts

### Q18: What is Little-Endian? Why does `0x53465331` appear as `31 53 46 53` in `xxd`?
> **Answer:** "x86 processors are little-endian, meaning the least significant byte is stored at the lowest memory address. `0x31` ('1') is the least significant byte of `0x53465331`, so it appears first on disk."

---

### Q19: Explain the C expression `bitmap[index / 8] |= (1u << (index % 8))`
> **Answer:** "`index / 8` selects which byte in the bitmap to access. `index % 8` selects which bit within that byte. `(1u << ...)` creates a single-bit mask, and `|=` (bitwise OR) sets that specific bit to 1 without changing the other 7 bits."

---

### Q20: Explain the difference between `fopen` modes `"wb+"`, `"rb+"`, and `"rb"`
> **Answer:**  
> - `"wb+"`: Creates/opens for read & write, **truncating the file to 0 bytes** (used in `simplefs_builder`).  
> - `"rb+"`: Opens for read & write **without truncating** existing contents (used in `simplefs_adder`).  
> - `"rb"`: Opens **read-only** (used for the user's source file to guarantee it is not modified).

---

### Q21: What do `fseek`, `ftell`, and `rewind` do?
> **Answer:** "`fseek` moves the file read/write cursor to a specific byte offset. `ftell` returns the current cursor position in bytes. `rewind` resets the cursor back to byte 0."

---

### Q22: Why does `inode_offset()` use `(inode_number - 1)`?
> **Answer:** "Because Inode numbers are 1-indexed ($1 \dots 32$), but C array offsets are 0-indexed ($0 \dots 31$). Inode 1 is at index 0 ($12288 + 0 \times 128$), Inode 2 is at index 1 ($12288 + 1 \times 128$)."

---

# TIER 4: Execution Logic & Critical Traps (Teacher Assessment)

### Q23: Why MUST the data bitmap bit be set INSIDE the allocation loop in `simplefs_adder.c`?
> **Answer:** "`find_free_data_block()` uses first-fit to find the first clear bit (`0`). If the bit is not set immediately, the next iteration sees the exact same clear bit and returns the same block number again, causing data corruption."

---

### Q24: Why does `simplefs_builder` zero-fill the entire disk before writing metadata?
> **Answer:** "It guarantees that all unused inodes, bitmap bits, and directory entries default cleanly to `0` (`free`), sets the exact disk size to 256 KiB, and prevents stale host memory from leaking onto the disk."

---

### Q25: Why does `find_free_directory_entry` start searching from index 2?
> **Answer:** "Slots 0 and 1 are permanently reserved for `.` and `..`. Searching from index 2 protects them from being overwritten by new files."

---

### Q26: Why are validation checks performed BEFORE allocating inodes and blocks?
> **Answer:** "To prevent **space leaks**. If we allocated blocks first and then rejected the file due to an invalid name or size, those blocks would remain permanently marked as used in the bitmap with no inode pointing to them."

---

### Q27: If a file is 5,000 bytes, how many blocks does it use, and what is stored in `inode.size`?
> **Answer:** "It uses **2 blocks** (Block 1: 4096 bytes; Block 2: 904 bytes + 3192 zero padding). `inode.size` stores the exact length: **`5000`**, not 8192."

---

### Q28: What happens when a 0-byte file is added?
> **Answer:** "It uses **0 data blocks** (`direct = {0,0,0}`), but still receives **1 inode** (`size = 0`, `type = 1`, `links = 1`) and **1 directory entry** in Block 4."

---

# TIER 5: Hex Dump (`xxd`) Whiteboard Questions

### Q29: The examiner shows you `0x03` at offset 4096 (Block 1). What does it mean?
> **Answer:** "`0x03` in binary is `0000 0011`. Bits 0 and 1 are set, meaning **Inode 1 (Root)** and **Inode 2 (First file)** are allocated."

---

### Q30: The examiner shows you `0x07` at offset 8192 (Block 2). What does it mean?
> **Answer:** "`0x07` in binary is `0000 0111`. Bits 0, 1, and 2 are set. Since $\text{Block} = \text{Index} + 4$, this means **Block 4 (Root)**, **Block 5**, and **Block 6** are allocated."

---

### Q31: Where on disk would you look to find the contents of Inode 3?
> **Answer:** "At byte offset $12288 + (3 - 1) \times 128 = 12288 + 256 = \mathbf{12544}$."

---

### Q32: Where on disk would you look to find the data of Block 5?
> **Answer:** "At byte offset $5 \times 4096 = \mathbf{20480}$."

---

## 5-Second Viva Memory Trigger

- **Image Size:** 256 KiB (64 blocks $\times$ 4096 bytes)
- **4 Offsets:** `0` (SB), `4096` (IB), `8192` (DB), `12288` (IT), `16384` (Data)
- **3 Struct Sizes:** Superblock = 36B, Inode = 128B, Dirent = 64B
- **Max Limits:** File size = 12288B (3 blocks), Name = 58 chars, Total files = 31
