---
layout: post
title: "OS Credential Dumping "
strapline: "T1003.008 - MITRE ATT&CK"
date: 2021-04-26 
published: true
lastmod: 2021-04-26
changefreq: monthly
priority: 0.5
categories: ATT&CK Atomic john auditd
excerpt_separator: <!--excerpt-->
---

In this blog post we investigate different ways to dump and decrypt Linux 
password files using John the Ripper which is described in the MITE Attack 
frameworks - Scheduled Task/Job: Systemd Timers - T1053.006 definition.

John the Ripper is an Open Source password security auditing and password 
recovery tool available for many operating systems. John the Ripper jumbo 
supports hundreds of hash and cipher types, including for: user passwords of 
Unix flavors (Linux, *BSD, Solaris, AIX, QNX, etc.), macOS, Windows, "web apps" 
(e.g., WordPress), groupware (e.g., Notes/Domino), and database servers 
(SQL, LDAP, etc.); network traffic captures (Windows network authentication, 
WiFi WPA-PSK, etc.); encrypted private keys (SSH, GnuPG, cryptocurrency wallets, 
etc.), filesystems and disks (macOS .dmg files and "sparse bundles", Windows 
BitLocker, etc.), archives (ZIP, RAR, 7z), and document files (PDF, Microsoft 
Office's, etc.) 

<!--excerpt-->

<br>

#### [From MITE Attack - OS Credential Dumping: /etc/passwd and /etc/shadow]()

> Adversaries may attempt to dump the contents of /etc/passwd and /etc/shadow to enable offline password cracking. Most modern Linux operating systems use a combination of /etc/passwd and /etc/shadow to store user account information including password hashes in /etc/shadow. By default, /etc/shadow is only readable by the root user.

> The Linux utility, unshadow, can be used to combine the two files in a format suited for password cracking utilities such as John the Ripper # /usr/bin/unshadow /etc/passwd /etc/shadow > /tmp/crack.password.db

<br>

## Techniques 

<br>

To test some techniques to dump and decrypt the /etc/shadow file we will use a 
Kali Linux distribution. On Linux, their are multiple ways to dump/view the 
contents of a file.

    cat /etc/shadow 
    grep .* /etc/shadow 
    head -1000 /etc/shadow
    nl /etc/shadow # nl - number lines of files
    tail -n 1000 /etc/shadow

    less /etc/shadow
    more /etc/shadow

<br>

    sudo unshadow /etc/passwd /etc/shadow > unshadow

    kali:$y$j9T$B4i9oW2LaERt/J5/X8bbN/$zzGfRqAZim/VofZcas3MhnfSdYddB5.zRulk087PN2A:1000:1000:Kali,,,:/home/kali:/usr/bin/zsh

    john --format=crypt unshadow

    john --show unshadow

    kali:kali:1000:1000:kali,,,:/home/kali:/usr/bin/zsh

<br>

## Detection

### Building auditd rules

    nano /etc/audit/rules.d/T1003.008.rules

    auditctl -w /etc/passwd -p rw -k privileged
    auditctl -w /etc/shadow -p rw -k privileged

    # -a always,exit -S all -F /etc/passwd -F perm=rw -k key=privileged
    # -a always,exit -S all -F /etc/shadow -F perm=rw -k key=privileged

<br>

### Finding the goodies

    :~# ausearch -ts today -i -k privileged
    :~# ausearch -ts today -i -k privileged |grep proctitle=.*
    :~# tail -f /var/log/audit/audit.log |grep key=\"privileged\"

<br>

## References

### man pages

- [man auditctl - a utility to assist controlling the kernel's audit system](/assets/text/man_auditctl.txt)
- [man ausearch - a tool to query audit daemon logs](/assets/text/man_ausearch.txt)
- [man john - a tool to find weak passwords of your users](/assets/text/man_john.txt)
- [man unshadow - combines passwd and shadow files](/assets/text/man_unshadow.txt)

<br>

### web pages

- [John the Ripper password cracker](https://www.openwall.com/john/)
- [John the Ripper FAQ](https://www.openwall.com/john/doc/FAQ.shtml)
- [MITRE ATT&CK OS Credential Dumping](https://attack.mitre.org/techniques/T1003/008/)
- [Atomic Red Team T1003.008](https://github.com/redcanaryco/atomic-red-team/blob/master/atomics/T1003.008/T1003.008.md)

<br>
<br>
