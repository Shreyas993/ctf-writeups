# OverTheWire Bandit: Levels 6-15

Notes for Levels 6 to 15 of Bandit. This set moves from searching the filesystem to text processing, decoding and compression, and finally SSH keys and network connections. Passwords are left out so the writeup doesn't spoil the game.

Each level is reached over SSH, using the password found in the previous level:

```
ssh bandit6@bandit.labs.overthewire.org -p 2220
```

---

## Level 6

**Task:** The password is stored somewhere on the server in a file owned by user `bandit7`, group `bandit6`, and exactly 33 bytes in size.

```
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
cat /var/lib/dpkg/info/bandit7.password
```

Searching from `/` covers the whole system, so most of the output would normally be "Permission denied" errors for folders I can't open. `2>/dev/null` throws the error output away and leaves only the match. The file turned out to be under `/var/lib/dpkg/info/`.

---

## Level 7

**Task:** The password is in `data.txt`, next to the word `millionth`.

```
grep millionth data.txt
```

`data.txt` is far too long to read, so `grep` prints just the line containing the word. The password is the second column.

---

## Level 8

**Task:** The password is the only line in `data.txt` that appears exactly once.

```
sort data.txt | uniq -u
```

`uniq -u` prints lines with no duplicates, but it only compares neighbouring lines. Sorting first puts identical lines next to each other, and the pipe passes the sorted output straight into `uniq`.

---

## Level 9

**Task:** The password is one of the few human-readable strings in `data.txt`, which is mostly binary, and it is preceded by several `=` characters.

```
strings data.txt | grep "=="
```

`strings` pulls out the readable text, and `grep` narrows it to the lines with the equals signs. Four lines matched, and the password was the last one, after `========== is`.

---

## Level 10

**Task:** The password is in `data.txt`, encoded in base64.

```
base64 -d data.txt
```

Base64 is an encoding, not encryption, so `-d` decodes it directly.

---

## Level 11

**Task:** The password is in `data.txt`, and every letter has been rotated by 13 places (ROT13).

```
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

`tr` replaces each letter in the first set with the matching one in the second. The second set is the alphabet starting from N, which undoes the shift.

---

## Level 12

**Task:** `data.txt` is a hexdump of a file that was compressed many times over. The password is inside the final layer.

First I made a working folder, since the home directory isn't writable, and reversed the hexdump:

```
mktemp -d
cp data.txt /tmp/tmp.xxxxxxxxxx
cd /tmp/tmp.xxxxxxxxxx
xxd -r data.txt > data.bin
file data.bin
```

From there it was a loop. I ran `file` to see what the current file was, unpacked it with the matching tool, and checked again. gzip and bzip2 won't unpack a file unless it has the right extension, so those needed a rename first:

```
mv data.bin data.gz && gzip -d data.gz        # gzip
mv data data.bz2 && bzip2 -d data.bz2         # bzip2
tar -xf data                                  # tar
```

The layers, in order, were gzip, bzip2, gzip, tar, tar, bzip2, tar, gzip, and then plain ASCII text with the password. That's nine rounds of `file` followed by the right command.

---

## Level 13

**Task:** There is no password for Level 14. The home directory holds a private SSH key, `sshkey.private`, to use instead.

My first attempt was to log in from inside the server:

```
ssh -i sshkey.private bandit14@localhost -p 2220
```

It was refused. The server blocks SSH connections from localhost to save resources, so the key has to be used from my own machine. I downloaded it with `scp`:

```
scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .
```

That failed at first with "Permission denied" because my PowerShell session opened in `C:\Windows\System32`, which isn't writable. After `cd ~` to move into my user folder, the download worked. Then I logged in with the key:

```
ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
cat /etc/bandit_pass/bandit14
```

No password prompt this time, because the key proves identity. Reading `/etc/bandit_pass/bandit14` worked because I was now logged in as `bandit14`.

---

## Level 14

**Task:** Submitting the current password to port 30000 on localhost returns the password for the next level.

```
nc localhost 30000
```

`nc` (netcat) opens a raw connection to a port. I pasted the Level 14 password, and the service replied with the next one.

---

## Level 15

**Task:** Same as Level 14, but on port 30001, which only accepts SSL/TLS.

```
openssl s_client -connect localhost:30001 -quiet
```

`nc` can't speak TLS, so `openssl s_client` is used to open an encrypted connection instead. It printed a self-signed certificate warning, which is expected here. I pasted the current password again and got back `Correct!` followed by the next one.

---

## Commands used

- `find`, `grep`: search for files and for lines inside files
- `sort`, `uniq`: put lines in order, then filter by repetition
- `strings`: extract readable text from binary files
- `base64`, `tr`: decode base64 and rotate letters back for ROT13
- `xxd`, `gzip`, `bzip2`, `tar`: reverse a hexdump and unpack compressed files and archives
- `file`: identify a file's type before choosing a tool
- `ssh -i`, `scp`: log in with a private key and copy files from the server
- `nc`, `openssl s_client`: connect to a port, with and without TLS
