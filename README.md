# UDP Service Recon & MD5 Hash-Cracking — CTF Write-up

> A hands-on lab walkthrough: discover a hidden **UDP** service with Nmap, talk to
> it with Netcat to retrieve an MD5 "voucher", then recover the unknown parts of
> the hashed plaintext with **crunch** + **hashcat**.

![Tools](https://img.shields.io/badge/tools-nmap%20%7C%20netcat%20%7C%20crunch%20%7C%20hashcat-blue)
![Platform](https://img.shields.io/badge/platform-Kali%20%2F%20Linux-557C94)
![License](https://img.shields.io/badge/license-MIT-brightgreen)

> [!NOTE]
> This is a lab exercise run against a provided target VM in a controlled,
> isolated environment. Placeholders such as `<STUDENT_ID>`, `<UBUNTU_IP>`,
> `<PORT>` and `<VOUCHER>` stand in for real values. Only perform this kind of
> scanning and cracking against systems you own or are authorized to test.

---

## The challenge

A target server (running on an Ubuntu VM) exposes a service on a **UDP** port in
the range `12345–12500`. Sending it your student ID returns a 32-character MD5
hash — a "voucher". The server computes that hash from:

```
plaintext = A || <STUDENT_ID> || B
          =  [letter][letter] <STUDENT_ID> [symbol][symbol]     (11 chars)
```

where **A** is two unknown lowercase letters and **B** is two unknown symbols. The
goal is to recover `A` and `B` by cracking the hash.

**Skills demonstrated:** UDP port scanning, raw service interaction, targeted
wordlist generation, mask/dictionary hash cracking, and verification.

---

## Environment

- **Attacker:** Kali Linux VM (nmap, netcat, crunch, hashcat)
- **Target:** Ubuntu VM running the challenge server
- Both VMs on the same virtual network (VirtualBox **NAT Network** or **Host-Only**
  adapter, *not* plain NAT), verified with `ping -c 3 <UBUNTU_IP>`.

---

## Step A — Find the open UDP port (Nmap `-sU`)

A normal TCP scan will never see a UDP service, so a UDP scan is essential:

```bash
sudo nmap -sU -p 12345-12500 --open -T4 <UBUNTU_IP>
```

| Flag | Meaning |
|------|---------|
| `-sU` | **UDP scan** |
| `-p 12345-12500` | restrict to the task's port range |
| `--open` | show only open ports |
| `-T4` | faster timing template |

UDP silence is ambiguous, so ports may show as `open|filtered`. Confirm with
version detection:

```bash
sudo nmap -sUV -p 12345-12500 --open -T4 <UBUNTU_IP>
```

➡️ The single open port is your `<PORT>`.

---

## Step B — Retrieve the voucher (Netcat `-u`)

Send the student ID to the service over UDP and read the reply:

```bash
echo -n "<STUDENT_ID>" | nc -u -w2 <UBUNTU_IP> <PORT>
```

| Flag | Meaning |
|------|---------|
| `-u` | UDP (must match the server) |
| `-w2` | wait 2 seconds for the reply, then quit |

If nothing comes back, the server may expect a trailing newline — retry without
`-n`:

```bash
echo "<STUDENT_ID>" | nc -u -w2 <UBUNTU_IP> <PORT>
```

➡️ The server returns a **32-hex-character MD5** — your `<VOUCHER>`.

---

## Step C — Recover A and B (crunch + hashcat)

Only **4 characters** are unknown (2 letters + 2 symbols) around a known student
ID, so the keyspace is roughly a million candidates — MD5 cracks it instantly.

**1. Save the target hash** (32 hex chars only, one line, no labels/whitespace):

```bash
echo "<VOUCHER>" > hash.txt
```

**2. Generate candidates with crunch** (`@` = lowercase letter, `^` = symbol;
digits are literal):

```bash
crunch 11 11 -t @@<STUDENT_ID>^^ -o wordlist.txt
```

**3. Crack with hashcat** (`-m 0` = MD5, `-a 0` = wordlist mode):

```bash
hashcat -m 0 -a 0 hash.txt wordlist.txt
# add --force if it complains about no real GPU in the VM
```

**Cross-check with a mask attack** (`?l` = lowercase, `?s` = symbol; no wordlist
file needed):

```bash
hashcat -m 0 -a 3 hash.txt ?l?l<STUDENT_ID>?s?s
```

**4. Show the recovered plaintext:**

```bash
hashcat -m 0 hash.txt --show      # prints <voucher>:<plaintext>
```

The plaintext looks like `xy1234567#@` → **A = first 2 chars**, **B = last 2
chars**.

**5. Verify by hand** (must reproduce the exact voucher):

```bash
echo -n "A<STUDENT_ID>B" | md5sum      # e.g. echo -n "xy1234567#@" | md5sum
```

---

## Troubleshooting

- **hashcat/crunch finds nothing** — make sure `hash.txt` contains *only* the 32
  hex characters (no `Voucher:` label, no stray spaces or newline).
- **Wrong hash?** The `md5sum` check in step 5 must match the voucher exactly.
- **Symbol outside crunch's default set** — the mask attack (`?s`) covers the full
  special-character set, so trust that result if the wordlist run misses.

---

## Key takeaways

- **UDP services are invisible to TCP scans** — always scan the right protocol.
- **Netcat is a universal probe** for raw TCP/UDP service interaction.
- **A small unknown keyspace collapses instantly** under MD5 — a reminder that
  MD5 is unsuitable for anything security-sensitive, and that predictable input
  structure massively shrinks a brute-force search.
- **crunch (targeted wordlist)** and **hashcat masks** are two routes to the same
  answer; using both cross-validates the result.

---

## Disclaimer

This write-up documents a **CSCI369 Ethical Hacking** lab performed against a
provided target in an isolated environment. It is shared for educational purposes.
Do not use these techniques against any system you do not own or are not
explicitly authorized to test.

## Author

**Ong Jun Han** — CSCI369 Ethical Hacking

## License

Released under the [MIT License](LICENSE).
