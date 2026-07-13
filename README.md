<h1 align="center">jasper</h1>

<p align="center">
  <img width="360" alt="jasper the cat" src="https://github.com/user-attachments/assets/4f20584c-8bf9-4684-9222-f53ab07c16cc" />
</p>

<p align="center">
  <img alt="shell" src="https://img.shields.io/badge/bash-5%2B-4EAA25?logo=gnubash&logoColor=white" />
  <img alt="license" src="https://img.shields.io/badge/license-MIT-blue" />
  <img alt="platform" src="https://img.shields.io/badge/platform-Kali%20Linux-557C94?logo=kalilinux&logoColor=white" />
</p>

<p align="center"><em>named after a cat.</em></p>

> **Only run this on stuff you're allowed to.** CTFs, HackTheBox, your own lab,
> or boxes you have written permission to test. Point it at random things on the
> internet and that's on you — it's illegal most places.

I got tired of typing the same 15 recon commands at the start of every box, so I
threw them all into one script. You give it an IP, it scans everything, figures
out the hostnames, sorts out `/etc/hosts` for you, and then runs the right tool
at whatever it finds — web, SMB, LDAP/AD, Kerberos, FTP, SNMP, databases, NFS,
RPC, WinRM, redis, the lot. It's all just standard Kali tools under the hood,
nothing weird to install.

Basically: run it, go make coffee, come back to a `SUMMARY.txt` that tells you
where to start.

## What it actually does

1. **Ports** — rustscan hits all 65535 TCP ports (falls back to `nmap -p-` if you
   don't have rustscan). There's a gentle top-1000 pass first so a rate-limiting
   box doesn't make you miss the obvious stuff.
2. **Services** — `nmap -sCV` on whatever's open. It reads TLS off the nmap
   service column, so HTTPS gets hit over `https://` and plain over `http://`,
   no dumb banner false-positives.
3. **Hostnames / domain** — pulls names out of TLS certs, HTTP redirects, and the
   AD scripts (`smb-os-discovery`, `rdp-ntlm-info`, LDAP rootDSE). **Run it with
   sudo and it writes the `IP hostname` lines straight into `/etc/hosts`** (backs
   it up first, won't double-add, warns you if something stale is shadowing your
   target). If it finds a dotted name like `active.htb` it uses that as the domain
   for vhost fuzzing and the `Host:` header. No sudo? It just prints the lines for
   you to paste.
4. **UDP** (sudo) — top-50 UDP for SNMP / DNS / SunRPC / TFTP. Runs in the
   background so it's not sitting there blocking everything else.
5. **Per-service enum** — all in parallel, one module per thing it finds:

   | Service | what it runs |
   |---------|------------------|
   | HTTP/S  | whatweb, headers/title/robots, feroxbuster for dirs, sslscan, wpscan (if it's WordPress), nuclei (with `-x`) |
   | SMB 139/445 | netexec null+guest shares, smbmap, smbclient `-L`, nmap smb scripts (ms17-010 included), enum4linux |
   | LDAP 389/636/3268 | anonymous rootDSE + naming-context dump (users/descriptions), `ldap-rootdse` |
   | Kerberos 88 | flags it as AD, tries an AS-REP roast, prints the kerberoast/userenum commands to run next |
   | FTP 21 | nmap `ftp-anon`, tries anonymous login |
   | SSH 22 | host key, algos, auth methods, `ssh-audit`, user-enum note |
   | telnet 23 | encryption/ntlm-info, banner, default-cred note |
   | VNC 5900/1 | `vnc-info`, realvnc auth-bypass, title |
   | SNMP 161/udp | onesixtyone community brute, snmpwalk public |
   | NFS/RPC 111/2049 | rpcinfo, showmount exports |
   | MSSQL / MySQL / Postgres / Mongo | nmap info + empty-password checks |
   | RDP 3389 | `rdp-ntlm-info` (leaks hostname/domain) |
   | WinRM 5985/6 | tells you the `evil-winrm` command |
   | redis 6379 | unauth `INFO` + `KEYS *` |
   | SMTP 25 | `smtp-commands`, open-relay, ntlm-info |
   | DNS 53 | AXFR zone transfer, ANY records |
   | finger 79 | user enumeration |

6. **Vhosts / subdomains** — ffuf `Host:`-header fuzz (auto-calibrated so a
   wildcard catch-all doesn't spam you) plus passive amass, once it knows the
   domain. Anything it finds gets dir-brute'd too, same as the main host, and it
   does it **at the same time** as the main scan — so you get `login`, `admin`,
   whatever, for the base domain *and* every vhost in one go.
7. **Got creds?** (`-u user -p pass`, or `-H hash` for pass-the-hash) — it re-runs
   enum *as that user* and sprays the creds across every open protocol with
   netexec (SMB/WinRM/LDAP/MSSQL/SSH/RDP/Postgres), dumps shares/users/groups, and
   roasts (AS-REP + Kerberoast) into `loot/`. If you see `Pwn3d!` that user is
   admin there.
8. **The good part** — a condensed `SUMMARY.txt` plus a color-coded **START HERE**
   list that greps everything for the stuff that matters (open shares, anon
   access, AS-REP hashes, ms17-010, community strings...) and ranks it CRIT/HIGH/
   INFO so you're not staring at a wall of output wondering what to do first.

Before any of that it does a quick **tool preflight** — anything you're missing
gets listed with what it would've done, so an empty file always means "found
nothing" and never "oh you didn't have the tool installed".

## Getting it

```bash
git clone https://github.com/ruwithma/jasper
cd jasper
chmod +x jasper
sudo ln -s "$PWD/jasper" /usr/local/bin/jasper   # optional, so it's on your PATH
```

Symlink it into `/usr/bin` too if you want `sudo jasper` to just work (that's the
one in sudo's `secure_path`).

## Using it

```
jasper <target> [options]        # run with sudo for /etc/hosts + UDP
```

| Option | what it does |
|--------|---------|
| `-d, --domain <name>` | AD/vhost base (auto-detected if you skip it) |
| `-o, --out <dir>`     | output dir (default `jasper-<target>`) |
| `-w, --wordlist <f>`  | dir-scan wordlist (default `dirb/common.txt`) |
| `-t, --threads <n>`   | web brute concurrency (default 50) |
| `-x, --nuclei`        | run nuclei on the web stuff |
| `-u, --user <name>`   | known username → authenticated re-enum + spray |
| `-p, --pass <pass>`   | known password (with `-u`) |
| `-H, --hash <lm:nt>`  | NTLM hash for pass-the-hash (with `-u`) |
| `--local-auth`        | creds are local accounts, not domain |
| `--no-udp`            | skip UDP |
| `--no-svc`            | skip per-service enum |
| `--no-web`            | skip web content enum |
| `--no-vhost`          | skip vhost fuzzing |
| `--deep`              | bigger wordlist (dirbuster medium) |
| `--quick`             | top-2000 TCP only, when you're in a hurry |

## Examples

```bash
sudo jasper 10.10.11.55            # the usual — everything, hosts sorted for you
sudo jasper 10.10.10.100 -x        # same but also run nuclei
jasper 10.10.10.161 --quick        # fast pass, no root
sudo jasper 10.10.11.55 -u bob -p 'S3cret!'        # once you've got creds
sudo jasper 10.10.10.100 -u admin -H :aad3b435...  # pass-the-hash
```

## Where it dumps everything

```
jasper-<target>/
├── SUMMARY.txt   the condensed findings + what to do next
├── scans/        rustscan, nmap tcp/udp/services (txt + xml)
├── services/     one file per service (smb.txt, 389_ldap.txt, ...)
├── web/          whatweb, headers, feroxbuster, sslscan, wpscan, nuclei
├── vhost/        ffuf vhost + passive amass
└── loot/         /etc/hosts backup + scratch
```

## Random notes

- Web fingerprint falls back to old TLS ciphers (`SECLEVEL=0`, TLS 1.0) so it
  still works on ancient CentOS / OpenSSL 1.0.2 boxes.
- Install `seclists` if you want bigger wordlists, then point `-w` at them.
- It only enumerates — it won't exploit anything for you. It just prints the
  follow-up commands (kerberoast, evil-winrm, psql, ...) so you know where to go.

## Contributing

PRs and issues welcome — have a look at [CONTRIBUTING.md](CONTRIBUTING.md). It's
just one Bash script, so keep it light and stick to the standard Kali toolset.

## License

[MIT](LICENSE) © ruwithma
