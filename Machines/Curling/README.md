# Curling

## User

### Scan
The first thing to do is scan the box. The output of `nmap -sC -sV 10.10.10.150` is:

```
Starting Nmap 7.70 ( https://nmap.org ) at 2019-03-07 21:42 EST
Nmap scan report for 10.10.10.150
Host is up (0.32s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8a:d1:69:b4:90:20:3e:a7:b6:54:01:eb:68:30:3a:ca (RSA)
|   256 9f:0b:c2:b2:0b:ad:8f:a1:4e:0b:f6:33:79:ef:fb:43 (ECDSA)
|_  256 c1:2a:35:44:30:0c:5b:56:6a:3f:a5:cc:64:66:d9:a9 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-generator: Joomla! - Open Source Content Management
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.33 seconds
```

### Get Admin Credentials
The website seems like a good place to start. Heading to `http://10.10.10.150` gives us an interesting homepage with a login window. Taking a hint from the box name, let's look at the page source with `curl 10.10.10.150` and look for interesting things. At the very bottom of the page is a comment: `secret.txt`. 

Curling 10.10.10.150/secret.txt gives us a strange set of characters: `Q3VybGluZzIwMTgh`. This is base64, so running `cat secret.txt | base64 --decode` gives us the decoded output of `Curling2018!`!

Now we have some sort of password, but where do we use it? The home page has a few posts on it so maybe we can find a username that goes with that password. Super User seems to be the only username on the page, but it doesn't work with the password. Hmmmmm....
The 'My first post of curling in 2018!' post signed by a user named Floris. Using Floris as the user and Curling2018! as the password works!

### Exploit Joomla
After doing some research on Joomla sites, it seems like it will be possible to upload some php code that will provide us with a reverse shell to the server. The administrator panel allows us to modify templates, so all we have to do is put a php script in one of the templates and then head to that site. Step-by-step, the process goes like this:

1. Navigate to `10.10.10.150/administrator` and log in
2. Go to Extensions > Templates > Templates
3. Choose a template and edit the `index.php` file
4. Paste the php script of your choosing (I used [this one](http://pentestmonkey.net/tools/php-reverse-shell/php-reverse-shell-1.0.tar.gz))
5. Make sure the script points at your HTB IP and an open port of your choosing (I used 1234)
6. Start a netcat listener on your machine with `nc -lnvp [PORT]`
7. Navigate to `10.10.10.150/templates/your_template_here/index.php`

### Reverse Password Backup
Now we have a shell! Unfortunately we're the `www-data` user so we can't do much. Although, with some prying we can exfil the `password_backup` file from the `/home/floris/` directory. Once we get that back on our machine we can start cracking the backup.

The backup looks like a hexdump, and `xxd` has a handy option to revert hexdumps back to binaries: `xxd -r password.hex`. Running `file` on the new output tells us that the file is `bzip2 compressed data, block size = 900k`. Decompressing this with `bzip2 -dk password.bz2` gives us another file that is `gzip compressed data, was "password"`. This can be decompressed with `gunzip`. This goes on for a couple more times and finally the password is revealed in .txt form: `5d<wdCbdZu)|hChXll`
 
Now that we have the user password, we can ssh into the box as floris, use our newfound password, and see `user.txt`!

## Root

### Exploit Curl
There is another interesting directory in `floris` that we have not looked at yet, `admin_area`. It contains a file named `input` with a url parameter set to localhost and a file named `report` with the html content from the locally hosted website. Taking another hint from the name of the box we can possibly deduce that this `input` file is being fed to some `curl` command that outputs the reponse to `report`. To test this theory, we can change the URL to something like `http://127.0.0.1/secret.txt` and see what happens. Shortly after, `report` changes to the output of `secret.txt` we saw earlier and our suspicions are confirmed.

With an `ls -l` we can see that the owner of these files is root!. Now we just have to figure out how to `curl` files from the root directory and we're set. Instead of setting the URL to the locally hosted website, we can just curl a file right off of the machine directly with `file:///root/root.txt`. Replacing the URL in `input` with this string and waiting a few seconds changes `report` to the contents of `root.txt`!
