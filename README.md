

# suid-lab

A hands-on lab demonstrating **SUID-bit privilege escalation via `mv`**.

It plants a root-owned `mv` with the SUID bit, abuses it to overwrite `/etc/passwd`,
and escalates to a UID-0 account. The point is to *understand* a common Linux
misconfiguration — and why `mv`/`cp`/`dd` should never carry the SUID bit.

> ⚠️ **Run only on a disposable VM you own.** This deliberately misconfigures
> permissions and rewrites `/etc/passwd`. A typo here can lock you out of login.
> Keep a second root shell open and back up `/etc/passwd` first.

## The idea

When the SUID bit is set on a root-owned executable, it runs with root's
**effective UID** regardless of who launches it. People's first instinct is to
"move a protected file somewhere readable" — but a same-filesystem `mv` is just
`rename(2)`, which preserves the inode's owner and mode, so the file stays
unreadable.

The real power is the opposite direction: an **arbitrary file-write** primitive.
A normal user can overwrite any file as root — here, `/etc/passwd` with a line
for a UID-0 account whose password hash sits directly in the second field
(so `/etc/shadow` is never consulted).

## Running it

Back up first:

```sh
sudo cp /etc/passwd /etc/passwd.bak
```

**1. Setup (once, as root)** — enable the plant step, build, then:

```sh
sudo ./suid_lab          # creates a root-owned SUID ./mv
```

**2. Exploit (as a normal user)** — disable the plant step, rebuild, then:

```sh
./suid_lab               # stages payload + lets the SUID mv write /etc/passwd
su thetechleadke         # password: linuxsucks123  ->  you are UID 0
```

## Cleanup

```sh
sudo cp /etc/passwd.bak /etc/passwd
rm -rf ./mv
```

## What to notice

- After the exploit, `ls -l /etc/passwd` shows a non-root owner — the same-fs
  `rename()` inherited the staged file's metadata. That artifact is exactly how
  this gets spotted in a real incident.
- Swap `mv` for `find`, `cp`, `dd`, `nano`, `vim`… on the same lab and watch which
  [GTFOBins](https://gtfobins.github.io/) techniques light up.

## Defensive takeaway

Audit SUID binaries regularly:

```sh
find / -perm -4000 -type f 2>/dev/null
```

Anything with a write path or shell escape (`mv`, `cp`, `dd`, editors, interpreters)
should **never** be SUID — a single arbitrary-write-as-root is full system compromise.

## Disclaimer

For education and defensive study on systems you own. Don't point it at anything else.
