# Linux Essentials — Git Bash Hands-On Project

> Nimbus Compute Academy — Module 2 · October 2026

Twenty hands-on Git Bash challenges covering the Linux fundamentals every DevOps engineer needs: navigation, file management, permissions, symbolic and hard links, archiving, searching, and recursive operations. Completed on Windows via Git Bash, with real cross-platform gotchas documented along the way as production-style postmortems.

---

## 🎯 What this project proves

- **Comfortable with the Linux shell**: `pwd`, `ls`, `cd`, `mkdir`, `touch`, `cp`, `mv`, `rm`, `chmod`, `ln`, `tar`, `find`, `grep`, `echo` with redirection
- **Understanding file-system fundamentals**: symbolic vs hard links, inodes, permission triplets, file types, hidden files
- **Reasoning about cross-platform differences**: Git Bash on Windows is not full Linux — knowing when and why fallback behaviors kick in
- **Postmortem discipline**: every error encountered is documented with root cause, resolution, and lesson learned

---

## 🔧 Environment setup (Windows + Git Bash)

Before running the challenges, two Windows-specific settings must be enabled or `ln -s` will silently create copies instead of real symlinks.

**1. Enable Windows Developer Mode**
Settings → search `developer` → **For developers** → toggle **Developer Mode: ON**

**2. Set strict symlink mode permanently in Git Bash**
```bash
echo 'export MSYS=winsymlinks:nativestrict' >> ~/.bashrc
source ~/.bashrc
```

Without BOTH of these, Git Bash's `ln -s` falls back to copying the file silently. See [postmortem #1](#postmortem-1--silent-symlink-fallback) for the full story.

---

## 📋 The 20 Challenges

### #1 — Print the present working directory
```bash
pwd
```

### #2 — Create and navigate into a directory
```bash
mkdir advanced_directory && cd advanced_directory
```

### #3 — Create two files and list with details
```bash
touch file_a.txt file_b.txt && ls -l
```

### #4 — Create a symbolic link
```bash
ln -s file_a.txt link_to_file_a
```
Verify with `ls -l link_to_file_a` → must start with `l` (not `-`).

### #5 — Set file permissions
```bash
chmod 644 file_b.txt
```
`6` = owner rw-, `4` = group r--, `4` = others r--.

### #6 — Create a subdirectory and copy files into it
```bash
mkdir nested_directory && cp file_a.txt file_b.txt nested_directory/
```

### #7 — Create a file with content in one command
```bash
cd nested_directory && echo "Hello World" > secret.txt
```
`&&` ensures the write only happens if `cd` succeeded (safer than `;`).

### #8 — Move multiple files to parent in one command
```bash
mv file_a.txt file_b.txt ../
```
Named files explicitly — a glob like `*.txt` would also match `secret.txt`.

### #9 — Create a hidden file
```bash
cd .. && touch .hidden_file
```
Leading `.` is the ONLY thing that marks a file "hidden" in Linux.

### #10 — List all files including hidden with human-readable sizes
```bash
ls -lah
```
`-l` long format · `-a` all including hidden · `-h` human-readable sizes.

### #11 — Archive all files (including hidden) into a subfolder
```bash
mkdir backup && tar --exclude='./backup' -cvf backup/archive.tar .
```
`--exclude='./backup'` prevents infinite recursion. See [postmortem #4](#postmortem-4--tar-cowardly-refusing-to-create-an-empty-archive).

### #12 — Extract an archive into a directory
```bash
mkdir restore && tar -xvf backup/archive.tar -C restore
```
`-x` extract · `-C restore` change to this directory before extracting.

### #13 — Create a hard link
```bash
touch temp.txt && ln temp.txt hard_link_to_temp
```
No `-s` = hard link. Both names share the same inode (same data).

### #14 — Find all .txt files recursively
```bash
find . -name "*.txt"
```
Quotes are critical — unquoted `*.txt` would be expanded by bash before find sees it.

### #15 — Delete a directory and all .txt files recursively in one command
```bash
rm -rf backup && find restore -name "*.txt" -delete
```
`find -delete` recurses into subdirectories; `rm restore/*.txt` would miss nested files.

### #16 — Create a symbolic link to a directory
```bash
ln -s restore link_to_restore
```
Hard links CANNOT point to directories (would create cycles). Only symlinks can.

### #17 — Find files modified in the last 7 days
```bash
find . -type f -mtime -7
```
`-7` = within last 7 days · `+7` = older than 7 days · `7` = exactly 7 days ago.

### #18 — Search for a string in all .txt files recursively
```bash
grep -r "Hello World" --include="*.txt" .
```
`--include` scopes the search to text files only (avoids binaries + speeds things up).

### #19 — Rename a file without changing directories
```bash
mv nested_directory/secret.txt nested_directory/hidden_secret.txt
```
`mv` handles both moving and renaming — same operation at the filesystem level.

### #20 — Copy a directory recursively
```bash
cp -r restore duplicate_restore
```
`-r` required for directories. For production backups, prefer `cp -a` (preserves metadata).

---

## 🔥 Postmortems

Production-style incident write-ups for the four significant errors encountered during the lab.

### Postmortem #1 — Silent symlink fallback

**Severity**: 🟡 Medium (silently incorrect results, not an error)
**Status**: ✅ Resolved
**Time lost**: ~45 min

**What happened**
`ln -s file_a.txt link_to_file_a` ran without error. `ls` showed the new entry. But `ls -l` revealed `-rw-r--r--` (regular file) instead of `lrwxrwxrwx` (symlink). Git Bash had **silently created a copy** of the target file.

**Root cause**
On Windows, creating NTFS symbolic links requires administrator elevation OR Windows Developer Mode enabled. Without this privilege, Git Bash's MSYS layer defaults to `MSYS=winsymlinks:native`, which falls back to copying the file rather than failing loudly.

**Resolution**
1. Enabled Windows Developer Mode in Settings → For developers
2. Set `export MSYS=winsymlinks:nativestrict` to force strict mode (fail loudly instead of silent copy)
3. Added that export to `~/.bashrc` so every future shell inherits it

**Verification**: `ls -l link_to_file_a` now shows `lrwxrwxrwx 1 PC 197121 10 Oct 8 12:58 link_to_file_a -> file_a.txt` — the `l` prefix and `->` arrow confirm a real symlink.

**Lessons learned**
- Silent fallbacks are more dangerous than loud failures. ALWAYS verify the type of what you created, not just that the command "succeeded without error."
- The `ls -l` first character is the authoritative file-type check (`l` for symlink, `-` for file, `d` for directory).
- For serious Linux work on Windows, WSL2 (Ubuntu) is cleaner than Git Bash. Added to backlog for next capstone.

---

### Postmortem #2 — Google steered me toward the wrong tool

**Severity**: 🟢 Low (educational)
**Status**: ✅ Resolved + rule adopted

**What happened**
When `ln -s` first failed, I Googled "symbolic link Git Bash on Windows." Top results pointed to the Windows-native `mklink` command (`cmd //c mklink link_to_file_a file_a.txt`). I used it and hit a DIFFERENT privilege error: *"You do not have sufficient privilege to perform this operation."*

**Root cause**
Both `ln -s` and `mklink` require the same Windows privilege. Switching tools didn't fix the privilege issue; it just changed the error message. More importantly, `mklink` is Windows-specific — practicing it in a Linux course trains the wrong muscle memory.

**Resolution**
Stuck with the Linux-native `ln -s`. Fixed the underlying privilege via Developer Mode (postmortem #1).

**Lessons learned**
- **For every Linux challenge, ask "what's the LINUX way?" BEFORE Googling.** Google's top results often optimize for the WHAT, not the WHY of the course.
- Argument order is REVERSED between the two:
  - `ln -s TARGET LINK_NAME` (Linux)
  - `mklink LINK_NAME TARGET` (Windows)
  - This single mismatch is the top source of symlink bugs when developers switch platforms.

---

### Postmortem #3 — Windows 11 native `sudo` + Git Bash don't play nicely

**Severity**: 🟢 Low
**Status**: ✅ Resolved via alternative path

**What happened**
After the silent symlink failure, I tried `sudo ln -s file_a.txt link_to_file_a` hoping to elevate just that command. First attempt returned *"Sudo is disabled on this machine. To enable it, go to the Developer Settings page in the Settings app"* — a surprise, since I didn't know Windows 11 24H2 had native sudo.

After enabling sudo in Settings → For developers, the command ran WITHOUT error, but `ls -l` still showed a copy, not a symlink.

**Root cause**
Windows 11's native `sudo` elevates a NEW process, but Git Bash's MSYS filesystem layer doesn't always inherit that elevation through the sudo boundary. The elevated child process ran successfully, but the symlink creation went through the un-elevated MSYS layer and fell back to copy mode.

**Resolution**
Enabled Windows Developer Mode (postmortem #1 fix) and used `MSYS=winsymlinks:nativestrict`. No sudo needed. The combination works because Developer Mode grants symlink privilege at the user level, so no per-command elevation is required.

**Lessons learned**
- For privilege-requiring operations in Git Bash on Windows, prefer "Run as administrator" on the shell itself, OR enable Developer Mode system-wide. Per-command `sudo` has subtle elevation-propagation gaps.
- Windows 11's native sudo is a quietly significant addition for Windows CLI developers.

---

### Postmortem #4 — tar "cowardly refusing to create an empty archive"

**Severity**: 🟢 Low
**Status**: ✅ Resolved — missing argument

**What happened**
```
$ tar --exclude='./backup' -cvf backup/archive.tar
tar: Cowardly refusing to create an empty archive
Try 'tar --help' or 'tar --usage' for more information.
```

**Root cause**
Missing the source argument at the end. tar received an output filename but no input to archive, so it refused rather than produce a useless 0-byte file.

**Resolution**
Added `.` at the end (= archive the current directory):
```bash
tar --exclude='./backup' -cvf backup/archive.tar .
```

**Lessons learned**
- tar's error messages are famously quirky but informative. *"Cowardly refusing"* is a GNU tool convention meaning "I'd rather fail loudly than produce useless output."
- Always check that you've specified WHAT to archive, not just WHERE to write.
- Read tool error messages carefully — many name the exact fix.

---

## 🧠 Overall Lessons Learned

1. **Verify, don't trust.** Commands returning without error don't mean they did what you wanted. `ls -l` tells the truth about file types; `tar -tvf` tells the truth about archive contents.

2. **The Linux way beats the Windows way, even on Windows.** Git Bash gives you Linux commands; use them. Fallbacks to native Windows tools train the wrong muscles for real DevOps work.

3. **`&&` beats `;` for safety.** Chain commands with `&&` so later commands only run if earlier ones succeeded. Prevents cascading errors like writing a file in the wrong directory.

4. **Scope your searches.** `grep -r --include` and `find -name` with proper quoting avoid both binary garbage and expansion bugs.

5. **Postmortems compound.** Documenting one error at the time saves hours of re-debugging the same issue weeks later.

---

## 📸 Screenshots

See [docs/screenshots/](docs/screenshots/) for terminal captures of key challenges:
- Challenge 4 — symlink verification (`lrwxrwxrwx`)
- Challenge 10 — `ls -lah` showing hidden file
- Challenge 11 — tar verbose output with hidden file included
- Challenge 14 — recursive find across subdirectories
- Challenge 18 — grep locating "Hello World" in nested file

---

## 👤 About

Built by **Tazoh Yanick Tazoh** ([@t-yanick](https://github.com/t-yanick)) — DevOps Engineer at Nimbus Compute (UK)

**Related work**: [nimbus-azure-capstone](https://github.com/t-yanick/nimbus-azure-capstone) — Two-region zero-trust Azure topology (Module 1 capstone).

**Connect**: [LinkedIn](https://www.linkedin.com/in/tazoh-yanick-tazoh/) · tyankee15@gmail.com

🇨🇲 → 🇨🇦

---

## 📄 License

MIT
