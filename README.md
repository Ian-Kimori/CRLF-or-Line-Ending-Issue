# CRLF-or-Line-Ending-Issue

# CRLF / Line Ending Issues
> Detection, diagnosis, and all fix methods — quick to advanced

---

## What It Is

| System       | Line Ending       | Bytes           |
| ------------ | ----------------- | --------------- |
| Windows      | `\r\n` (CRLF)     | `0x0D 0x0A`     |
| Linux / Kali | `\n` (LF only)    | `0x0A`          |

The hidden `\r` (carriage return) shows as `^M` in `cat -A` and breaks parsers like nmap, ffuf, dirsearch.

---

## Step 1 — Detect

**1. Reveal hidden characters**
```bash
cat -A ips.txt
```
`^M` at end of each line = CRLF confirmed

**2. File type check (script use)**
```bash
file ips.txt
```
Output containing `CRLF line terminators` = Windows file

---

## Step 2 — Fix (choose one)

### ✅ 1. `dos2unix` — cleanest one-liner *(recommended)*
Modifies file in-place.
```bash
dos2unix ips.txt
```

### 2. `sed` — always available
No external tool needed. In-place strip.
```bash
sed -i 's/\r$//' ips.txt
```

### 3. `tr` — safe output to new file
Non-destructive: writes to `clean.txt`.
```bash
tr -d '\r' < ips.txt > clean.txt
```

---

## Step 3 — Harden Your Script

### Option A — Pre-clean whole file at script top *(preferred)*
Runs once. Cleaner, faster than per-line stripping.
```bash
sed -i 's/\r$//' "$input_file"
```

### Option B — Per-line strip inside loop
Adds minor overhead but keeps original file untouched.
```bash
target=$(echo "$raw_target" | tr -d '\r')
```

### Option C — Detect-then-fix (smart guard)
Skips processing if file is already clean.
```bash
if file "$input_file" | grep -q CRLF; then
    echo "Fixing Windows line endings..."
    dos2unix "$input_file"
fi
```

### Option D — Full input sanitize *(recommended combo)*
Strip CR + collapse whitespace. Put both at script top + in loop.
```bash
# Top of script
sed -i 's/\r$//' "$input_file"

# Inside read loop
target=$(echo "$target" | tr -d '\r' | xargs)
```

---

## Fix at Source (Long-Term)

| Tool          | How                                                              |
| ------------- | ---------------------------------------------------------------- |
| VS Code       | Bottom-right status bar → click `CRLF` → change to `LF`        |
| Vim           | `:set fileformat=unix`                                           |
| Git global    | `git config --global core.autocrlf input`                       |
| .editorconfig | `end_of_line = lf`                                               |

---

## Where This Appears in Pentesting

- Wordlists (ffuf / dirsearch)
- Scope files (bug bounty)
- IP / CIDR target lists
- Password lists
- API endpoint lists
- Burp Suite imports

> Many "tool failures" — *invalid input*, *no targets*, *parse error* — are actually input formatting problems.

---

## Mental Model — When a Tool Throws Errors

Always suspect:

1. **Encoding** — UTF-8 vs UTF-16 vs Latin-1
2. **Line endings** — CRLF vs LF (`cat -A` to check)
3. **Hidden characters** — null bytes, BOM, trailing spaces
