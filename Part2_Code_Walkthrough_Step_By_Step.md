# PART 2 — Step-by-Step Code Walkthrough & Architecture
**CSE321 Lab Term Project — SimpleFS**  
**Focus:** Complete breakdown of execution pipelines, subtle mechanics, and number conversions.

---

# 1. `simplefs_builder.c` — The 8-Step Formatting Pipeline

`simplefs_builder` is the program that formats a blank disk image from scratch into a valid SimpleFS filesystem.

```
+-----------------------------------------------------------------------------------+
| THE 8 STEPS OF simplefs_builder                                                   |
+-----------------------------------------------------------------------------------+
| 1. Parse CLI arguments (--image <disk_name>)                                      |
| 2. Create disk file and write 64 zero-filled blocks (262,144 bytes)               |
| 3. Populate Superblock struct (36 bytes) & write to Block 0 (offset 0)            |
| 4. Set bit 0 in Inode Bitmap & write 4096 bytes to Block 1 (offset 4096)          |
| 5. Set bit 0 in Data Bitmap & write 4096 bytes to Block 2 (offset 8192)           |
| 6. Populate Root Inode struct (128 bytes) & write to Block 3 (offset 12288)       |
| 7. Populate '.' and '..' Dirents (64B each) & write to Block 4 (offset 16384)     |
| 8. Close file pointer and report success                                          |
+-----------------------------------------------------------------------------------+
```

---

### Step-by-Step Deep-Dive of `simplefs_builder.c`

#### Step 1: Parse Command-Line Options
```c
if (argc != 3) { printf("Usage: %s --image <image_name>\n", argv[0]); return 1; }
if (strcmp(argv[1], "--image") != 0) { printf("Error: expected --image option.\n"); return 1; }
image_name = argv[2];
```
- Validates that the user passed exactly 3 arguments and the flag is `--image`.

#### Step 2: Open and Zero-Fill the Entire 256 KiB Image
```c
fp = fopen(image_name, "wb+");
for (int i = 0; i < TOTAL_BLOCKS; i++) {
    fwrite(zero_block, BLOCK_SIZE, 1, fp);
}
```
- **Why zero-fill first?**
  1. Sets the exact file size on disk to $64 \times 4096 = 262,144\text{ bytes}$ (256 KiB).
  2. Guarantees that every unused inode, bitmap bit, and directory slot defaults cleanly to `0` (`free`).
  3. Prevents old garbage memory on your hard drive from leaking into the disk image.

#### Step 3: Populate and Write Superblock (Block 0)
```c
memset(&sb, 0, sizeof(sb));
sb.magic = 0x53465331;        /* 'SFS1' */
sb.block_size = 4096; sb.total_blocks = 64; sb.inode_count = 32;
sb.inode_bitmap_block = 1; sb.data_bitmap_block = 2;
sb.inode_table_block = 3;  sb.data_region_block = 4;
sb.root_inode = 1;

fseek(fp, SUPERBLOCK_BLOCK * BLOCK_SIZE, SEEK_SET); /* Offset 0 */
fwrite(&sb, sizeof(sb), 1, fp);
```
- Records the layout so future programs know where every region begins.

#### Step 4: Mark Root Inode in Inode Bitmap (Block 1)
```c
set_bit(inode_bitmap, 0); /* Inode 1 is bit index 0 */
fseek(fp, INODE_BITMAP_BLOCK * BLOCK_SIZE, SEEK_SET); /* Offset 4096 */
fwrite(inode_bitmap, BLOCK_SIZE, 1, fp);
```
- Sets bit 0 of byte 0 to `1` (`0x01`). Marks Inode 1 as in-use.

#### Step 5: Mark Root Data Block in Data Bitmap (Block 2)
```c
set_bit(data_bitmap, 0); /* Root entries live in Block 4 -> index 0 */
fseek(fp, DATA_BITMAP_BLOCK * BLOCK_SIZE, SEEK_SET); /* Offset 8192 */
fwrite(data_bitmap, BLOCK_SIZE, 1, fp);
```
- Sets bit 0 of byte 0 to `1` (`0x01`). Marks Block 4 as in-use.

#### Step 6: Populate and Write Root Inode (Block 3)
```c
memset(&root_inode, 0, sizeof(root_inode));
root_inode.type = TYPE_DIRECTORY;       /* 2 */
root_inode.links = 2;                    /* '.' and '..' */
root_inode.size = 2 * sizeof(dirent_t); /* 128 bytes */
root_inode.direct[0] = ROOT_DATA_BLOCK;  /* Block 4 */

fseek(fp, inode_offset(ROOT_INODE), SEEK_SET); /* Offset 12288 */
fwrite(&root_inode, sizeof(root_inode), 1, fp);
```

#### Step 7: Create `.` and `..` Directory Entries (Block 4)
```c
/* '.' entry */
dot.inode_no = ROOT_INODE; dot.type = TYPE_DIRECTORY;
strncpy(dot.name, ".", 58);

/* '..' entry */
dotdot.inode_no = ROOT_INODE; dotdot.type = TYPE_DIRECTORY;
strncpy(dotdot.name, "..", 58);

fseek(fp, ROOT_DATA_BLOCK * BLOCK_SIZE, SEEK_SET); /* Offset 16384 */
fwrite(&dot, sizeof(dot), 1, fp);       /* Byte 16384 */
fwrite(&dotdot, sizeof(dotdot), 1, fp); /* Byte 16448 */
```

#### Step 8: Close and Finish
```c
fclose(fp);
```

---

# 2. `simplefs_adder.c` — The 16-Step File Addition Pipeline

`simplefs_adder` injects an external file (e.g. `test1.txt`) into an existing SimpleFS disk image.

```
+-----------------------------------------------------------------------------------+
| THE 16 STEPS OF simplefs_adder                                                    |
+-----------------------------------------------------------------------------------+
|  1. Open image read/write ("rb+")                                                 |
|  2. Read Superblock & verify magic number (0x53465331)                            |
|  3. Open source file read-only ("rb")                                             |
|  4. Measure source file size using fseek(SEEK_END) and ftell()                    |
|  5. Check file_size <= 12288 bytes (max 3 blocks); reject if too big              |
|  6. Check strlen(source_name) <= 58 chars; reject if too long                     |
|  7. Calculate required_blocks = (file_size + 4095) / 4096 (0 for 0-byte file)     |
|  8. Scan Root Directory (Block 4) for duplicate names; reject if exists (FIONA)   |
|  9. Read Inode Bitmap (Block 1) and find first free inode (SOLAIMAN)              |
| 10. Read Data Bitmap (Block 2) and allocate blocks first-fit (SOLAIMAN)           |
| 11. Copy source file data into allocated blocks with zeroed buffer (SOLAIMAN)     |
| 12. Write new file's inode_t to Inode Table in Block 3 (SOLAIMAN)                 |
| 13. Write updated Inode Bitmap and Data Bitmap back to disk (SOLAIMAN)            |
| 14. Find first free directory slot in Block 4 starting from index 2 (FIONA)       |
| 15. Write new dirent_t into Block 4 at slot offset (FIONA)                        |
| 16. Increase root_inode.size by +64 bytes and rewrite to Block 3 (FIONA)          |
+-----------------------------------------------------------------------------------+
```

---

### Why Validation (Steps 5, 6, 8) Happens BEFORE Allocation (Steps 9, 10)

> **Golden Rule of File System Design:**  
> **"Validate everything first, commit changes last."**

If we allocated Inode 2 and Data Block 5 **first**, and then realized the filename was longer than 58 characters or was a duplicate:
- Exiting with an error would leave Inode 2 and Block 5 marked as `1` in the bitmaps.
- Since no file references them, those blocks are permanently lost.
- This is called a **Space Leak** (or disk leak).

---

# 3. The Two Most Subtle Code Mechanics (Guaranteed Viva Questions!)

---

### Subtle Mechanic 1: Setting the Bitmap Bit INSIDE the Allocation Loop

Look at lines 115–125 of `simplefs_adder.c`:

```c
for (int index = 0; index < required_blocks; index++) {
    int blk = find_free_data_block(data_bitmap);
    if (blk == -1) { /* error */ }
    allocated_blocks[index] = blk;
    set_bit(data_bitmap, data_bitmap_index(blk)); /* <-- CRITICAL: MUST BE HERE */
}
```

#### What happens if `set_bit` is outside the loop?
`find_free_data_block()` is a **first-fit** algorithm: it starts at index 0 and returns the first bit that equals `0`.
1. Iteration 1: finds bit 1 is `0` $\to$ returns **Block 5**.
2. Iteration 2: bit 1 is STILL `0` (not set yet!) $\to$ returns **Block 5 again**.
3. Iteration 3: returns **Block 5 again**.

A 3-block file would receive `{5, 5, 5}`. Block 5 is overwritten 3 times, destroying file contents!

Setting `set_bit()` **inside** the loop marks Block 5 as used immediately, so iteration 2 gets Block 6, and iteration 3 gets Block 7.

---

### Subtle Mechanic 2: Re-Zeroing the Buffer on Every Iteration

Look at lines 133–139 of `simplefs_adder.c`:

```c
for (int index = 0; index < required_blocks; index++) {
    unsigned char buf[BLOCK_SIZE];
    memset(buf, 0, BLOCK_SIZE);                    /* <-- CRITICAL: MUST BE HERE */
    fread(buf, 1, BLOCK_SIZE, source);
    fseek(image, (long)allocated_blocks[index] * BLOCK_SIZE, SEEK_SET);
    fwrite(buf, BLOCK_SIZE, 1, image);             /* Full 4096 bytes written */
}
```

#### Example: Adding a 5,000-byte file
- **Iteration 1 (Block 5):** `fread()` reads 4096 bytes $\to$ buffer completely filled $\to$ writes 4096 bytes.
- **Iteration 2 (Block 6):** `fread()` reads only the remaining 904 bytes ($5000 - 4096 = 904$).
  - **Without `memset`:** Bytes 904 to 4095 still contain data from Block 5! Stale garbage gets written to disk.
  - **With `memset`:** Bytes 904 to 4095 are clean `0x00` padding.

---

# 4. The 4 Number Conversions (Easy Arithmetic Reference)

There are 4 different numbers used in SimpleFS. Keep them straight:

| Name | Range | Meaning | Example |
|---|---|---|---|
| **Inode Number** | $1 \dots 32$ | 1-indexed identifier used in dirents and user code | `Inode 2` |
| **Inode Table Index** | $0 \dots 31$ | 0-indexed position in inode array on disk | `Index 1` |
| **Data Bitmap Index** | $0 \dots 59$ | 0-indexed bit in data bitmap (Block 2) | `Index 1` |
| **Absolute Block Number** | $4 \dots 63$ | The physical block address on disk ($0 \dots 63$) | `Block 5` |

---

### The Conversion Formulas:

$$\text{Inode Table Index} = \text{Inode Number} - 1$$
$$\text{Absolute Block Number} = \text{Data Bitmap Index} + 4$$
$$\text{Data Bitmap Index} = \text{Absolute Block Number} - 4$$
$$\text{Byte Offset of Block } N = N \times 4096$$
$$\text{Byte Offset of Inode } N = 12288 + (N - 1) \times 128$$

---

# 5. Quick Viva Practice for Part 2

> **Examiner:** *"Walk me through what happens when `simplefs_adder` adds a 10 KB file."*  
> **Your Answer:**  
> 1. *"First, it validates the image magic number `0x53465331`."*  
> 2. *"It checks that 10 KB $\le 12288$ bytes and name $\le 58$ chars."*  
> 3. *"It scans Block 4 to ensure the name is not a duplicate."*  
> 4. *"It calculates $\lceil 10240 / 4096 \rceil = 3$ required blocks."*  
> 5. *"It finds the first free inode (e.g. Inode 2) and three free blocks (Blocks 5, 6, 7) using first-fit, setting each bitmap bit inside the loop."*  
> 6. *"It copies the file block-by-block using zeroed 4096-byte buffers."*  
> 7. *"It writes Inode 2 with `size = 10240` and `direct = {5, 6, 7}`."*  
> 8. *"It finds an empty directory slot in Block 4 starting from index 2, writes the 64-byte directory entry, and adds $+64$ to `root_inode.size`."*
