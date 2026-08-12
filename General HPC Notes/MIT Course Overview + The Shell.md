# The Shell

## Basics

- Most platforms provide some kind of shell (e.g. Windows PowerShell).
- The most common shell is **Bash** (Bourne Again Shell).
- Bash is almost its own programming language.
- Arguments are separated by whitespace. Use quotation marks or a backslash `\` to include whitespace in an argument — precision matters.
- The shell knows where programs are through something called the **PATH environment variable**.

## Notable Commands

| Command | What It Does |
|---|---|
| `echo` | Prints something to the screen |
| `echo $PATH` | Prints your current PATH |
| `which <cmd>` | Shows the full path to a command |
| `pwd` | Prints the current working directory |
| `ls -l` | Lists files with detailed info (permissions, size, etc.) |
| `mv` | Rename or move a file |
| `cp` | Copy a file (takes source path, then destination path) |
| `rm` | Remove a file (not recursive by default on Linux) |
| `mkdir` | Create a new directory |
| `tail` | Prints the last line(s) of output |
| `ctrl+l` | Clear the terminal |
| `xdg-open <file>` | Open a file in the appropriate program |

## File System and Paths

- A **path** is a way to name a file's location on a computer.
- **Windows**: one root per partition (`C:/`, `D:/`). **Linux/macOS**: everything lives under `/`.
- **Absolute path**: fully describes where a file is, starting from root.
- **Relative path**: relative to your current location (`pwd`).
- Special directories: `.` means the current directory, `..` means the parent directory.
- `~` always refers to your home directory. `cd -` switches to the previous directory.
- Most programs run relative to the current working directory.

## Permissions

`ls -l` shows permissions in a format like `-rwxr-xr-x`. A `d` prefix indicates a directory.

### For files

| Permission | Meaning |
|---|---|
| `r` (read) | Can read the file |
| `w` (write) | Can modify the file |
| `x` (execute) | Can run the file as a program |

### For directories

| Permission | Meaning |
|---|---|
| `r` (read) | Can list the contents of the directory |
| `w` (write) | Can create, rename, or move files inside it |
| `x` (execute / "search") | Can enter (`cd` into) the directory |

## I/O Redirection and Pipes

By default the shell has an **input stream** (keyboard) and an **output stream** (screen). You can redirect both.

| Operator | Effect |
|---|---|
| `< file` | Use a file as input instead of the keyboard |
| `> file` | Send output to a file (overwrites) |
| `>> file` | Send output to a file (appends) |
| `cmd1 \| cmd2` | Pipe: send output of `cmd1` as input to `cmd2` |

`tail` prints the last line of a command's output — useful at the end of a pipe.

## User Privileges

- The **root user** (user ID `0`) is the administrator and can do anything on the system.
- `sudo` lets you run a single command as root.
- `sudo su` opens a full shell as the superuser.
- `/sys` is a special filesystem exposing kernel parameters — you can read and modify device behavior directly from the shell.
- The `#` symbol at the shell prompt indicates you are running as root.

---

## Related Notes

- [[Shell Tools and Scripting]] — scripting and advanced shell usage building on these basics
- [[Linux refresher]] — Linux concepts that go alongside shell skills
- [[HPC Intro]] — the cluster environment where these shell skills are used
