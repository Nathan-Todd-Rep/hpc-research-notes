# Shell Tools and Scripting

## Variables

Define a variable with `foo=bar` (no spaces around `=`). Access it with `$foo`.

```bash
foo=bar
echo "$foo"   # prints: bar
echo '$foo'   # prints: $foo
```

Double-quoted strings expand variables (like f-strings in Python). Single-quoted strings are always literal.

## Functions

```bash
mcd () {
    mkdir -p "$1"
    cd "$1"
}
```

This function creates a directory and immediately `cd`s into it. `$1` is the first argument passed to the function.

## Special Variables

| Variable | Meaning |
|---|---|
| `$0` | Script name |
| `$1` – `$9` | Arguments to the script |
| `$@` | All arguments |
| `$#` | Number of arguments |
| `$?` | Return code of the last command |
| `$$` | Process ID of the current script |
| `!!` | The entire last command |
| `$_` | Last argument of the last command |

## Exit Codes and Conditionals

- Commands report results through **STDOUT** (output), **STDERR** (errors), and a **return code**.
- Return code `0` = success. Anything else = an error occurred.
- `true` always returns `0`. `false` always returns `1`.

```bash
false || echo "Oops, fail"       # prints: Oops, fail
true  || echo "Won't print"      #
true  && echo "Things went well" # prints: Things went well
false && echo "Won't print"      #
true  ; echo "This always runs"  # prints: This always runs
false ; echo "This always runs"  # prints: This always runs
```

| Operator | Behavior |
|---|---|
| `&&` | Run second command only if first succeeded |
| `\|\|` | Run second command only if first failed |
| `;` | Always run both, regardless of exit codes |

## Command Substitution

`$(CMD)` executes `CMD` and substitutes its output inline:

```bash
for file in $(ls); do
    echo "$file"
done
```

`<(CMD)` executes `CMD`, saves its output to a temporary file, and substitutes that filename:

```bash
diff <(ls foo) <(ls bar)   # compare directory contents without temp files
```

## Example Script

```bash
#!/bin/bash

echo "Starting program at $(date)"
echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
    grep foobar "$file" > /dev/null 2> /dev/null
    # grep exits with 1 if pattern not found; we discard both stdout and stderr
    if [[ $? -ne 0 ]]; then
        echo "File $file does not have any foobar, adding one"
        echo "# foobar" >> "$file"
    fi
done
```

## Wildcards and Globbing

Use `?` and `*` to match filenames by pattern. Given files `foo`, `foo1`, `foo2`, `foo10`, `bar`:

| Pattern | Matches |
|---|---|
| `rm foo?` | Deletes `foo1`, `foo2` (exactly one character) |
| `rm foo*` | Deletes `foo`, `foo1`, `foo2`, `foo10` (any characters) |

## Curly Brace Expansion

Use `{}` to expand a common substring across multiple values:

```bash
convert image.{png,jpg}
# expands to: convert image.png image.jpg

cp /path/to/project/{foo,bar,baz}.sh /newpath

mv *{.py,.sh} folder        # move all .py and .sh files

touch {foo,bar}/{a..h}      # creates foo/a...foo/h and bar/a...bar/h
diff <(ls foo) <(ls bar)    # shows: < x  ---  > y
```

## `find` Command

Recursively search for files matching criteria:

```bash
find . -name src -type d                        # all directories named src
find . -path '*/test/*.py' -type f              # .py files inside test folders
find . -mtime -1                                # files modified in the last day
find . -size +500k -size -10M -name '*.tar.gz'  # zip files in a size range

find . -name '*.tmp' -exec rm {} \;             # delete all .tmp files
find . -name '*.png' -exec magick {} {}.jpg \;  # convert PNG to JPG
```

## Shell History

```bash
history | grep find   # show all past commands containing "find"
```

---

## Related Notes

- [[MIT Course Overview + The Shell]] — the shell fundamentals this scripting builds on
- [[Linux refresher]] — Linux commands used throughout shell scripts
