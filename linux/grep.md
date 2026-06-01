# `grep` Cheat Sheet - Searching in Files

## Basic Syntax

```bash
grep [options] pattern [file...]
grep "word" file.txt
grep "word" file1.txt file2.txt
```

---

## Common Options

```bash
-i          # Case-insensitive match
-v          # Invert match (lines that do NOT match)
-n          # Show line numbers
-c          # Count matching lines
-l          # List only filenames with matches
-L          # List only filenames WITHOUT matches
-r          # Recursive search in directories
-R          # Recursive, follows symlinks
-w          # Match whole words only
-x          # Match whole lines only
-q          # Quiet mode (exit code only, no output)
-s          # Suppress error messages
```

---

## Output Control

```bash
-o          # Print only the matched part
-h          # Suppress filename prefix
-H          # Always print filename prefix
--color     # Highlight matches
-m 5        # Stop after 5 matches
```

---

## Context Lines

```bash
-A 3        # 3 lines After match
-B 3        # 3 lines Before match
-C 3        # 3 lines before and after (Context)
```

---

## Regex Modes

```bash
grep  "pattern"   # Basic regex (BRE) - default
grep -E "pattern" # Extended regex (ERE) - same as egrep
grep -P "pattern" # Perl-compatible regex (PCRE)
grep -F "pattern" # Fixed string (no regex) - same as fgrep
```

---

## Pattern Examples

```bash
grep "^start"          # Lines starting with "start"
grep "end$"            # Lines ending with "end"
grep "^$"              # Empty lines
grep "c.t"             # c + any char + t  (cat, cut, cot…)
grep "colou\?r"        # BRE: optional 'u'  (color or colour)
grep -E "colou?r"      # ERE: same
grep -E "cat|dog"      # Either "cat" or "dog"
grep -E "^(foo|bar)"   # Lines starting with foo or bar
grep -E "[0-9]{3}"     # Three consecutive digits
grep -E "\b\w{5}\b"    # Exactly 5-letter words
grep -P "\d{4}-\d{2}"  # PCRE: date-like pattern
```

---

## Recursive & File Filtering

```bash
grep -r "TODO" .                        # Search all files under current dir
grep -r "TODO" . --include="*.py"       # Only .py files
grep -r "TODO" . --exclude="*.log"      # Exclude .log files
grep -r "TODO" . --exclude-dir=".git"   # Exclude a directory
```

---

## Multiple Patterns

```bash
grep -e "foo" -e "bar" file.txt         # Match foo OR bar
grep -f patterns.txt file.txt           # Read patterns from file
```

---

## Practical Combos

```bash
grep -rn "TODO" . --include="*.js"      # Find TODOs with line numbers in JS files
grep -v "^#" config.conf                # Strip comments
grep -c "" file.txt                     # Count all lines (like wc -l)
grep -rl "secret" /etc/                 # List files containing "secret"
grep -i "error" app.log | grep -v "404" # Errors excluding 404s
grep -Po '(?<=user=)\w+' file.txt       # Extract value after "user="
```

---

## Exit Codes

```bash
0   # Match found
1   # No match
2   # Error (bad option, file not found, etc.)
```

```bash
grep -q "pattern" file && echo "found" || echo "not found"
```