# OverTheWire Bandit: Levels 0-5

Bandit is a beginner wargame on OverTheWire that teaches the Linux command line through a series of small puzzles. These are my notes for Levels 0 to 5. Passwords are left out so the writeup doesn't spoil the game.

Each level is reached over SSH, using the password found in the previous level:

```
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

---

## Level 0

**Task:** Log in and find the password for Level 1, stored in a file called `readme` in the home directory.

```
ls
cat readme
```

`ls` listed the home directory and `cat` printed the file, which contained the password.

---

## Level 1

**Task:** The password is in a file named `-`.

```
ls
cat ./-
```

Running `cat -` would hang, because a lone dash is read as "take input from the keyboard". Prefixing the name with `./` makes it an explicit path, so `cat` opens it as a file.

---

## Level 2

**Task:** The password is in a file named `--spaces in this filename--`.

```
cat "./--spaces in this filename--"
```

This needed two fixes at once. The spaces would split the name into separate arguments, so the name is quoted. The leading dashes would be read as options, so `./` goes in front again. Tab completion handles both automatically.

---

## Level 3

**Task:** The password is in a hidden file inside the `inhere` directory.

```
cd inhere
ls
ls -a
cat ./...Hiding-From-You
```

Plain `ls` returned nothing. `ls -a` listed `.`, `..` and `...Hiding-From-You`. I had expected a hidden file to start with a single dot, but this one starts with three. The first two entries are not files, they refer to the current and parent directories, so the third was the one to read.

---

## Level 4

**Task:** The password is in the only human-readable file in `inhere`, out of ten files named `-file00` to `-file09`.

```
cd inhere
file ./*
cat ./-file07
```

`file ./*` reports the content type of every file. Nine were raw data and `-file07` was ASCII text, so that was the one to open. The `./` prefix is needed again because of the leading dash.

---

## Level 5

**Task:** The password is in a file somewhere under `inhere`. It is human-readable, exactly 1033 bytes, and not executable. There are too many subdirectories to check by hand.

```
cd inhere
find . -type f -size 1033c ! -executable
cat ./maybehere07/.file2
```

`find` filtered the whole tree by file type (`-type f`), size (`-size 1033c`, where `c` means bytes) and permissions (`! -executable`). It returned a single match, a hidden file inside `maybehere07`.

---

## Commands used

- `ls`, `ls -a`: list files, with `-a` including hidden ones
- `cat`: print a file
- `cd`: change directory
- `file`: show a file's type
- `find`: search by type, size and permissions
