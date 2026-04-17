# Input File Issues — Complete Reference
> CRLF, missing newlines, encoding, BOM, and hidden characters — detection, diagnosis, and fixes

---

## What It Is

| Issue | Description | Symptom |
|---|---|---|
| CRLF line endings | Windows uses `\r\n`, Linux expects `\n` | `^M` in `cat -A`, tools get malformed args |
| Missing trailing newline | File ends without `\n` | Last line silently skipped by bash `while read` |
| BOM (Byte Order Mark) | UTF-8/UTF-16 BOM prefix | First token corrupted, tools reject file |
| UTF-16 encoding | 2-byte-per-char encoding | Looks like binary, `xxd` shows `ff fe` at offset 0 |
| Null bytes | `\x00` embedded in file | Breaks most parsers silently |
| Trailing whitespace | Spaces/tabs after content | Variables carry hidden whitespace, matching fails |

| System | Line Ending | Bytes |
|---|---|---|
| Windows | `\r\n` (CRLF) | `0x0D 0x0A` |
| Linux / Kali | `\n` (LF only) | `0x0A` |

---

## Phase 1 — Detect

### 1. Reveal hidden characters
Run before touching any input file.

```bash
cat -A filename.txt
```

| Output | Meaning |
|---|---|
| `172.29.75.0/24^M$` | CRLF confirmed — Windows line endings |
| `172.29.75.0/24$` | Clean LF — but check for trailing newline |
| `172.29.75.0/24` | No `$` at end — missing trailing newline |

### 2. File type check
```bash
file filename.txt
```

| Output | Meaning |
|---|---|
| `ASCII text, with CRLF line terminators` | Windows file — needs fixing |
| `ASCII text` | Clean Unix file |
| `UTF-16 Unicode text` | Wrong encoding — needs `iconv` |
| `data` | Binary or null bytes present |

### 3. Hex dump — deepest inspection
Use when `cat -A` is ambiguous, or when you suspect null bytes, BOM, or UTF-16.

```bash
xxd filename.txt | head -10
```

| Bytes at position | Meaning |
|---|---|
| `0d 0a` at line ends | CRLF |
| `ff fe` at offset 0 | UTF-16 LE BOM — breaks everything |
| `ef bb bf` at offset 0 | UTF-8 BOM — breaks some tools |
| `00` interspersed | Null bytes |
| `0a` only | Clean LF |

### 4. Missing trailing newline check
Bash `while IFS= read -r` silently drops the last line if the file has no final `\n`.

```bash
tail -c 1 filename.txt | xxd
```

| Output | Meaning |
|---|---|
| `0a` | Trailing newline present — safe |
| anything else | No trailing newline — last line will be skipped |

### 5. Isolate variable value in script
Before fixing the file, confirm your script receives what you think. Brackets make empty strings and whitespace visible.

```bash
# Put this inside your while loop temporarily
echo "TARGET: [$target]"

# Or test the pipeline directly
alive=$(sudo nmap -sn -n 172.29.75.0/24 | grep "Nmap scan report" | awk '{print $NF}')
echo "HOSTS: [$alive]"
```

| Output | Meaning |
|---|---|
| `TARGET: [172.29.75.0/24\r]` | CRLF leaking into variable |
| `HOSTS: []` | Sweep returning nothing — check permissions or CRLF |
| `TARGET: [172.29.75.0/24]` | Clean |

---

## Phase 2 — Diagnose

Match your symptom to the root cause before choosing a fix.

| Symptom | Likely Cause | Go To |
|---|---|---|
| Script runs, prints "done", no output files | Missing trailing newline (only line skipped) | Step 8 |
| Script runs, no output files | CRLF in target variable → malformed nmap arg | Step 7 |
| Tool says "invalid input" / parse error | CRLF in wordlist, BOM at start of file | Steps 7, 9 |
| Only the last line is never processed | Missing trailing newline | Step 8 |
| Works manually, fails in script | CRLF invisible in terminal but corrupts variable | Step 5, then 7 |
| Tool output looks garbled / binary | UTF-16 encoding | Step 9 |
| First item always fails, rest work | UTF-8 BOM on first line | Step 9 |
| Passwords / usernames not matching despite correct values | Trailing whitespace or CRLF in wordlist | Steps 7, Option D |

---

## Phase 3 — Fix

### Fix CRLF line endings

**Option 1 — `dos2unix` (recommended)**
Cleanest, modifies in-place.
```bash
dos2unix filename.txt
```

**Option 2 — `sed` (always available)**
No external tool needed.
```bash
sed -i 's/\r$//' filename.txt
```

**Option 3 — `tr` (non-destructive)**
Writes to a new file, original untouched.
```bash
tr -d '\r' < filename.txt > clean.txt
```

**Option 4 — detect then fix (smart guard)**
Skips processing if file is already clean.
```bash
if file "$input_file" | grep -q CRLF; then
    echo "Fixing Windows line endings..."
    dos2unix "$input_file"
fi
```

---

### Fix missing trailing newline

```bash
# Append a blank line
echo "" >> filename.txt

# Or: add newline only if missing (idempotent)
sed -i -e '$a\' filename.txt
```

Verify:
```bash
cat -A filename.txt
# Every line including the last should end with $
```

---

### Fix BOM and encoding

```bash
# Strip UTF-8 BOM
sed -i '1s/^\xef\xbb\xbf//' filename.txt

# Convert UTF-16 to UTF-8
iconv -f UTF-16 -t UTF-8 input.txt > output.txt

# Strip null bytes
tr -d '\000' < filename.txt > clean.txt
```

---

## Phase 4 — Harden Your Script

Apply these permanently so file origin never matters.

### Option A — Pre-clean whole file at script top (preferred)
Runs once. Faster than per-line stripping.
```bash
sed -i 's/\r$//' "$input_file"
```

### Option B — Per-line strip inside loop
Keeps original file untouched.
```bash
target=$(echo "$raw_target" | tr -d '\r')
```

### Option C — Detect then fix (smart guard)
```bash
if file "$input_file" | grep -q CRLF; then
    echo "Fixing Windows line endings..."
    dos2unix "$input_file"
fi
```

### Option D — Full input sanitize (recommended combo)
Strip CR + collapse whitespace. Apply at both top of script and inside the loop.
```bash
# Top of script
sed -i 's/\r$//' "$input_file"

# Inside read loop
target=$(echo "$target" | tr -d '\r' | xargs)
```

### Option E — Defensive while-read pattern (permanent fix)
The `|| [[ -n "$raw" ]]` clause catches the final line even when no trailing newline exists. This is the correct long-term fix — the file state no longer matters.

```bash
while IFS= read -r raw || [[ -n "$raw" ]]; do
    target=$(echo "$raw" | tr -d '\r' | xargs)
    [[ -z "$target" ]] && continue
    # your logic here
done < "$input_file"
```

> Combining Option A (pre-clean) with Option E (defensive loop) means your script handles any file origin without manual intervention.

---

## Fix at Source (Long-Term)

Prevent the problem from occurring in the first place.

| Tool | How |
|---|---|
| VS Code | Bottom-right status bar → click `CRLF` → change to `LF` |
| Vim | `:set fileformat=unix` then `:w` |
| Git global | `git config --global core.autocrlf input` |
| `.editorconfig` | `end_of_line = lf` |
| Nano | Save as-is on Linux — nano preserves existing line endings |

---

## When to Run Each Check

| Situation | What to run |
|---|---|
| Any new input file before first use | `cat -A` |
| File came from Windows (Teams, email, USB, Notepad) | `cat -A`, then `dos2unix` |
| File created in Excel or Word (CSV exports etc.) | `file` + `xxd` — may have BOM or CRLF |
| Tool produces no output with no error | `cat -A` + variable isolation test (step 5) |
| Works manually, not in script | Variable isolation test (step 5) |
| Credentials/wordlist not matching despite being correct | `cat -A` for CRLF, `xxd` for trailing whitespace or null bytes |
| First item always fails, rest work | `xxd` — suspect BOM |
| Downloaded wordlist from GitHub on Windows | `dos2unix` preemptively |
| Writing a new reusable script | Add Option E (defensive loop) as default pattern |

---

## Where This Appears in Pentesting

- IP / CIDR target lists (nmap, masscan)
- Wordlists (ffuf, dirsearch, gobuster)
- Password and username lists (hydra, medusa, crackmapexec)
- Scope files (bug bounty)
- API endpoint lists
- Burp Suite imports
- Custom exploit scripts reading targets from file
- SSH known_hosts and authorized_keys (BOM breaks key parsing)

> Many "tool failures" — *invalid input*, *no targets*, *parse error*, *zero results* — are input formatting problems, not tool bugs.

---

## Mental Model — Triage Order When a Tool Fails

When a tool throws an error or silently produces nothing, check in this order:

1. **Permissions** — does the tool need root? (`sudo`)
2. **Line endings** — CRLF vs LF (`cat -A`)
3. **Trailing newline** — last line being skipped (`tail -c 1 | xxd`)
4. **Encoding** — UTF-16, UTF-8 BOM (`file`, `xxd`)
5. **Hidden characters** — null bytes, trailing spaces (`xxd`, `cat -A`)
6. **Variable isolation** — is the value actually what you think? (echo with brackets)
