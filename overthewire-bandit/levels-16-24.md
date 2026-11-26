# OverTheWire Bandit: Levels 16-24

Notes for Levels 16 to 24 of Bandit. This set covers port scanning, reading cron jobs and exploiting them, a setuid binary, and brute-forcing a small keyspace. Passwords are left out so the writeup doesn't spoil the game.

Each level is reached over SSH, using the password found in the previous level:

```
ssh bandit16@bandit.labs.overthewire.org -p 2220
```

---

## Level 16

**Task:** The password for the next level comes from one port between 31000 and 32000 on localhost. Most ports in that range just echo input back, and only one speaks SSL and returns real credentials.

```
nmap -p 31000-32000 localhost
nmap -sV -p 31046,31518,31691,31790,31960 localhost
```

The first scan found five open ports. The second, with `-sV`, probed each one to identify the service. Two came back as `ssl/...`, but only one of them, port 31790, was flagged as unrecognized rather than a known echo service, and its fingerprint output showed it was prompting for a password. Connecting confirmed it:

```
openssl s_client -connect localhost:31790 -quiet
```

Pasting the current password returned `Correct!` followed by a full private SSH key rather than a plain password, for logging into Level 17.

---

## Level 17

**Task:** Use the private key from Level 16 to log into bandit17, then find the password by comparing two files.

The key had to be saved on the server first (home directories aren't writable for new files directly in the usual way, so I used a temp folder), then copied down to my own machine with `scp`, since OverTheWire blocks SSH connections that originate from itself:

```
scp -P 2220 bandit16@bandit.labs.overthewire.org:/tmp/tmp.xxxxxxxxxx/sshkey17.private .
ssh -i sshkey17.private bandit17@bandit.labs.overthewire.org -p 2220
```

Once logged in:

```
diff passwords.old passwords.new
```

`diff` printed the one line that changed between the two files, marked with `<` for the old version and `>` for the new one. The new line was the password for Level 18.

---

## Level 18

**Task:** Logging into bandit18 normally logs you straight back out, because `.bashrc` has been altered to do that. The password is in a `readme` file in the home directory.

```
ssh bandit18@bandit.labs.overthewire.org -p 2220 cat readme
```

Appending a command to the end of the `ssh` call runs that one command on the server and prints its output before the forced logout happens, so the interactive session never actually needs to open.

---

## Level 19

**Task:** A setuid program, `bandit20-do`, runs any command as user `bandit20`.

```
ls -l bandit20-do
./bandit20-do cat /etc/bandit_pass/bandit20
```

The permissions showed `-rwsr-x---`, with an `s` in place of the owner's normal execute bit. That `s` is the setuid flag, meaning the program always runs as its owner (`bandit20`) regardless of who starts it, so it could read a password file that was otherwise off limits to me.

---

## Level 20

**Task:** A program called `suconnect` connects to a given port on localhost, reads one line, and checks it against the current password. If it matches, it sends back the next one.

```
echo 'CURRENT_PASSWORD' | nc -l -p 61234 &
./suconnect 61234
```

I started my own listener on port 61234 with `nc -l`, set to hand out the current password to whatever connects, with `&` keeping it running in the background. Then I ran `suconnect` pointed at that same port, and it connected to my listener, read the password, confirmed the match, and printed the next one straight to my terminal.

---

## Level 21

**Task:** A cron job runs a script on a schedule. Read the job and the script to see what it does.

```
ls /etc/cron.d/
cat /etc/cron.d/cronjob_bandit22
cat /usr/bin/cronjob_bandit22.sh
```

The cron entry ran the script every minute as user `bandit22`. The script itself copied `/etc/bandit_pass/bandit22` into a fixed file in `/tmp` and set its permissions to `644` (world-readable). Since cron re-ran it every minute, the file was already sitting there to read.

```
cat /tmp/t7O6lds9S0RqQh9aMcz6ShpAoZKF7fgv
```

---

## Level 22

**Task:** Same idea as Level 21, but the output file's name isn't fixed. It's generated from an MD5 hash of a sentence containing the username.

```
cat /etc/cron.d/cronjob_bandit23
cat /usr/bin/cronjob_bandit23.sh
```

The script ran `whoami` to get its own username (`bandit23`, from the cron entry), built the sentence `I am user bandit23`, hashed it with `md5sum`, and used that hash as the output file name. I couldn't run the script as `bandit23` myself, but I could reconstruct the same file name by running the identical pipeline with the username typed in directly:

```
cat /tmp/$(echo I am user bandit23 | md5sum | cut -d ' ' -f 1)
```

---

## Level 23

**Task:** A cron job watches a folder, runs any file in it that is owned by `bandit23`, then deletes everything in the folder, all as user `bandit24`.

```
cat /etc/cron.d/cronjob_bandit24
cat /usr/bin/cronjob_bandit24.sh
```

Reading the script showed it looped over every file in `/var/spool/bandit24/foo`, checked the owner was `bandit23`, ran it with a 60-second timeout, and deleted it afterward. Since I was logged in as `bandit23`, any script I dropped there would be executed as `bandit24`, including reading files I couldn't otherwise access. I wrote a small script to copy the Level 24 password into a file I controlled:

```
cd $(mktemp -d)
chmod 777 .
echo '#!/bin/bash' > getpass.sh
echo "cat /etc/bandit_pass/bandit24 > $(pwd)/password.txt" >> getpass.sh
touch password.txt
chmod 666 password.txt
chmod 777 getpass.sh
cp getpass.sh /var/spool/bandit24/foo/
```

After waiting about a minute for cron to pick it up:

```
cat password.txt
```

---

## Level 24

**Task:** A service on port 30002 expects the current password and a 4-digit PIN on one line, space-separated. Wrong PINs just get rejected; the right one returns the next password. With only 10,000 possible PINs, trying them all is quick.

```
cd $(mktemp -d)
for i in $(seq -w 0000 9999); do echo "CURRENT_PASSWORD $i"; done > pins.txt
cat pins.txt | nc -q 1 localhost 30002 > results.txt
grep -v "Wrong" results.txt
```

`seq -w 0000 9999` generated every PIN with leading zeros kept, and the loop paired each one with the password in its own file. Piping the whole file into `nc -q 1` sent every line in a single connection, closing it a second after the input ended, and saved every reply to a file. Filtering out the lines containing "Wrong" left just the one that mattered, which confirmed the correct PIN and handed back the Level 25 password.

---

## Commands used

- `nmap`, `nmap -sV`: scan for open ports, then identify the service on each
- `openssl s_client`: connect to a port over SSL/TLS
- `scp`: copy a file to or from a remote server
- `diff`: compare two files and show what changed
- `ssh user@host command`: run a single command on a remote server without an interactive session
- `ls -l`: check file permissions, including the setuid bit
- `nc -l`: run a listener on a local port
- `cron`, reading `/etc/cron.d/*` and the scripts they call
- `whoami`, `md5sum`, `cut`, and `$(...)` command substitution: rebuild a generated file name
- `seq -w`, loops, and `nc -q`: generate and send many inputs to brute-force a small keyspace
