# Phase 1 - Day 1 Lab Report: VFS & Link Mechanics

## 📌 Objective
To investigate the Linux Virtual File System (VFS) abstraction layer by analyzing how the kernel manages file metadata (inodes), directory entries (dentries), and link counters when files are created and deleted.

---

## 🛠️ Lab Execution Logs & Analysis

### Step 1: Initial Infrastructure Setup
A clean project workspace was initialized, and a base file containing raw string data was created.
```bash
mkdir -p ~/DEVOPS-300/phase1/day1
cd ~/DEVOPS-300/phase1/day1
echo "Deep dive linux data" > original.txt
```

### Step 2: Creating Links
A hard link (`hardlink.txt`) and a symbolic link (`softlink.txt`) were generated targeting the base file.
```bash
ln original.txt hardlink.txt
ln -s original.txt softlink.txt
```

### Step 3: Metadata and Inode Inspection
The system metadata state was verified using the `ls -i` and `stat` commands.

#### Terminal Output Observations:
* **`original.txt` Inode:** `3178560` | **Links Count:** `2`
* **`hardlink.txt` Inode:** `3178560` | **Links Count:** `2`
* **`softlink.txt` Inode:** `3178570` | **Links Count:** `1`

#### Critical Engineering Insight:
The `original.txt` and `hardlink.txt` files share the exact same inode number (`3178560`). This proves that a hard link is not a copy of a file; it is simply an additional directory entry (dentry) pointing to the same underlying physical metadata structure on the disk. Conversely, `softlink.txt` has a unique inode (`3178570`) and its data payload size is exactly `12` bytes, which represents the string length of its text target path: `"original.txt"`.

---

## 💥 The Destruction Phase (Deletion Analysis)

The primary file descriptor was unlinked from the filesystem:
```bash
rm original.txt
```

### Post-Deletion Verification Results:

1. **`cat hardlink.txt` -> Result: `Deep dive linux data`**
   * **Why it worked:** When `original.txt` was removed, the kernel decremented the inode `3178560` link counter from `2` down to `1`. Because the counter was greater than `0`, the kernel did not wipe or release the data blocks. The data remains fully intact and accessible through the remaining hard link.
   
2. **`cat softlink.txt` -> Result: `cat: softlink.txt: No such file or directory`**
   * **Why it failed:** The soft link's inode (`3178570`) is still completely intact. However, because its internal text pointer references the literal string path `"original.txt"`, and that path's dentry mapping was destroyed during the `rm` command, it resolves to nothing. This creates a "dangling" or "broken" symlink.

---

## 🧠 Core Engineering Takeaways
* **Files are anonymous:** In Linux, a file's name does not live inside its data blocks or its inode. File names live strictly inside directory structures (dentries) mapping text strings to inode integers.
* **Garbage Collection:** Data blocks on storage media are only marked as free/unallocated by the filesystem when the absolute final hard link counter (`Links: 0`) drops to zero and no active running processes hold an open file descriptor to it.
