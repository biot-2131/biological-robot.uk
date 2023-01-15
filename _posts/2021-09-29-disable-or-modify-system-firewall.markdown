---
layout: post
title: "Disable or Modify System Firewall"
strapline: "T1562.004 - MITRE ATT&CK"
date: 2021-09-29 
published: true
lastmod: 2021-09-29
changefreq: monthly
priority: 0.5
categories: ATT&CK Atomic ufw auditd
excerpt_separator: <!--excerpt-->
---

### [Description from ATT&CK](https://attack.mitre.org/techniques/T1562/004/)

Adversaries may disable or modify system firewalls in order to bypass controls limiting network usage. Changes could be disabling the entire mechanism as well as adding, deleting, or modifying particular rules. This can be done numerous ways depending on the operating system, including via command-line, editing Windows Registry keys, and Windows Control Panel.

Modifying or disabling a system firewall may enable adversary C2 communications, lateral movement, and/or data exfiltration that would otherwise not be allowed.

<!--excerpt-->

<br>

## Configuration 

To build a set of controls to detect the activities described in MITRE ATT&CK's T1562.004 - Disable or Modify System Firewall we will enable Ubuntu's 20.04 LTS Uncomplicated Firewall (UFW) and use the Linux Audit daemon auditd to detect and record any changes.

### ufw rules 

    Status: active
    Logging: on (medium)
    Default: deny (incoming), allow (outgoing), deny (routed)
    New profiles: skip

    To                         Action      From
    --                         ------      ----
    22                         ALLOW IN    10.0.0.0/28 


### auditd rules

    auditctl -w /etc/default/ufw -p wa -k UFW_config
    auditctl -w /etc/ufw -p wa -k UFW_rules
    auditctl -w /usr/sbin/iptables -p xa -k UFW_iptables
    auditctl -w /usr/sbin/ufw -p xa -k UFW_ufw
    auditctl -w /var/log/ufw.log -p r -k UFW_log

<br>

## Controls and Tests

### 1: Stop/Start Uncomplicated Firewall (UFW)

Command:

    ufw disable

Log: 

    PROCTITLE . 09:25:28 . /usr/sbin/ufw disable 
    EXECVE .    09:25:28 . argc=3 a0=/usr/bin/python3 a1=/usr/sbin/ufw . 
    SYSCALL .   09:25:28 . syscall=execve . pid's uid's comm=ufw exe=/usr/bin/python3.8 . key=UFW_ufw 

    PROCTITLE . 09:25:28 . proctitle=iptables -F ufw-logging-deny 
    SYSCALL .   09:25:28 . syscall=setsockopt . pid's uid's comm=iptables exe=/usr/sbin/xtables-legacy-multi  
    470+ iptables rules 

Command: 

    ufw enable

Log:

    PROCTITLE . 09:25:30 . /usr/sbin/ufw enable 
    EXECVE .    09:25:30 . argc=3 a0=/usr/bin/python3 a1=/usr/sbin/ufw . 
    SYSCALL .   09:25:30 . syscall=execve . pid's uid's comm=ufw exe=/usr/bin/python3.8 . key=UFW_ufw 

    PROCTITLE . 09:25:30 . proctitle=iptables-restore -n 
    SYSCALL .   09:25:30 . syscall=setsockopt . pid's uid's comm=iptables-restor exe=/usr/sbin/xtables-legacy-multi  
    300+ iptables rules 

<br>

The UFW_ufw detected the execution of ufw and the UFW_iptables the execution of iptables as it dropped/created all of the tables. 

Note: UFW_iptables is quite noisy creating numerous entries. However it is the only rule that will detect the use of systemctl to stop the iptables service, so the verbosity has to be tolerated.

<br>

### 2: Stop/Start UFW firewall using systemctl

Command: 

    systemctl stop ufw
    
Log:

    PROCTITLE . 09:25:36 . proctitle=iptables -F ufw-logging-deny 
    SYSCALL .   09:25:36 . comm=iptables exe=/usr/sbin/xtables-legacy-multi  
    NETFILTER_CFG 09:25:36 . table=filter . comm=iptables 
    130+ iptables rules

Command:

    systemctl start ufw
    
Log:

    PROCTITLE 09:25:38 . proctitle=iptables-restore -n 
    SYSCALL 09:25:38 . comm=iptables-restor exe=/usr/sbin/xtables-legacy-multi 
    NETFILTER_CFG 09:25:38 . table=filter . comm=iptables-restor 
    170+ iptables rules

<br>

The UFW_iptables detected the execution of iptables it dropped/created tables, see note above on verbosity. 

<br>

### 3: Turn off/on UFW logging

Command: 

    ufw logging off
    
Log: 

    PROCTITLE . 09:25:43 . /usr/sbin/ufw logging off 
    EXECVE .    09:25:43 . argc=4 a0=/usr/bin/python3 a1=/usr/sbin/ufw . 
    SYSCALL .   09:25:43 . syscall=chmod . pid's uid's comm=ufw exe=/usr/bin/python3.8 . key=UFW_rules 
    160+ iptables entries

Command:

    ufw logging low
    
Log:

    PROCTITLE . 09:25:45 . /usr/sbin/ufw logging low 
    EXECVE .    09:25:45 . argc=4 a0=/usr/bin/python3 a1=/usr/sbin/ufw . 
    SYSCALL .   09:25:45 . syscall=execve . pid's uid's comm=ufw exe=/usr/bin/python3.8 . key=UFW_ufw 
    190+ iptables rules

Command:

    ufw status verbose
    
Log:

    PROCTITLE . 09:25:46 . /usr/sbin/ufw status verbose 
    EXECVE .    09:25:46 . argc=4 a0=/usr/bin/python3 a1=/usr/sbin/ufw . 
    SYSCALL .   09:25:46 . syscall=execve . pid's uid's comm=ufw exe=/usr/bin/python3.8 . key=UFW_ufw 

<br>

The UFW_ufw detects the execution of the ufw commands and the UFW_rules detects the changes to the /etc/ufw/ufw.conf file.

<br>

### 4: Add and delete UFW firewall rules

Command:

    ufw prepend deny from 1.2.3.4
    
Log:

    PROCTITLE . 09:25:50 . /usr/sbin/ufw prepend deny from 1.2.3.4 
    EXECVE .    09:25:50 . argc=6 a0=/usr/bin/python3 a1=/usr/sbin/ufw . a4=from a5=1.2.3.4 
    SYSCALL .   09:25:50 . syscall=execve . pid's uid's comm=ufw exe=/usr/bin/python3.8 . key=UFW_ufw 
    80+ iptables entries 

Command:

    ufw status numbered
    
Log:

    PROCTITLE . 09:25:51 . /usr/sbin/ufw status numbered 
    EXECVE .    09:25:51 . argc=4 a0=/usr/bin/python3 a1=/usr/sbin/ufw . 
    SYSCALL .   09:25:51 . syscall=execve . pid's uid's comm=ufw exe=/usr/bin/python3.8 . key=UFW_ufw 

Command:

    { echo y; echo response; } | ufw delete 1
    
Log:

    PROCTITLE . 09:25:53 . /usr/sbin/ufw delete 1 
    SYSCALL .   09:25:53 . syscall=chmod . pid's uid's comm=ufw exe=/usr/bin/python3.8 . key=UFW_rules 
    PROCTITLE . 09:25:53 . proctitle=/usr/sbin/iptables -D ufw-user-input -s 1.2.3.4 -j DROP 
    SYSCALL .   09:25:53 . syscall=setsockopt . pid's uid's comm=iptables exe=/usr/sbin/xtables-legacy-multi  
    30+ iptables entries 

<br>

The UFW_ufw detects the execution of the ufw commands and the UFW_rules detects the changes to the /etc/ufw/user.rules file.

<br>

### 5: Edit UFW firewall user.rules file

Command:

    echo "# THIS IS A COMMENT" >> /etc/ufw/user.rules
    
Log:

    PROCTITLE . 09:25:57 . proctitle=/bin/bash /code/bsh/art/art_test.bsh /code/bsh/art/atomics/T1562.004.yaml y a 
    SYSCALL .   09:25:57 . syscall=openat . pid's uid's comm=art_test.bsh exe=/usr/bin/bash . key=UFW_rules 

Command:

    sed -i 's/# THIS IS A COMMENT//g' /etc/ufw/user.rules
    
Log:

    PROCTITLE . 09:25:59 . proctitle=sed -i s/# THIS IS A COMMENT//g /etc/ufw/user.rules 
    SYSCALL . 09:25:59 . syscall=rename . pid's uid's comm=sed exe=/usr/bin/sed . key=UFW_rules 

<br>

The UFW_rules detects the changes to the /etc/ufw/user.rules file.

<br>

### 6: Edit UFW firewall ufw.conf file

Command:

    echo "# THIS IS A COMMENT" >> /etc/ufw/ufw.conf
    
Log:

    PROCTITLE . 09:26:02 . proctitle=/bin/bash /code/bsh/art/art_test.bsh /code/bsh/art/atomics/T1562.004.yaml y a 
    SYSCALL .   09:26:02 . syscall=openat . pid's uid's comm=art_test.bsh exe=/usr/bin/bash . key=UFW_rules 

Command:

    sed -i 's/# THIS IS A COMMENT//g' /etc/ufw/ufw.conf
    
Log:

    PROCTITLE . 09:26:04 . proctitle=sed -i s/# THIS IS A COMMENT//g /etc/ufw/ufw.conf 
    SYSCALL .   09:26:04 . syscall=rename . pid's uid's comm=sed exe=/usr/bin/sed . key=UFW_rules 

<br>

The UFW_rules detects the changes to the /etc/ufw/ufw.conf file.

<br>

### 7: Edit UFW firewall sysctl.conf file

Command:

    echo "# THIS IS A COMMENT" >> /etc/ufw/sysctl.conf
    
Log: 

    PROCTITLE . 09:26:08 . proctitle=/bin/bash /code/bsh/art/art_test.bsh /code/bsh/art/atomics/T1562.004.yaml y a 
    SYSCALL .   09:26:08 . syscall=openat . pid's uid's comm=art_test.bsh exe=/usr/bin/bash . key=UFW_rules 

Command:

    sed -i 's/# THIS IS A COMMENT//g' /etc/ufw/sysctl.conf
    
Log:

    PROCTITLE . 09:26:10 . proctitle=sed -i s/# THIS IS A COMMENT//g /etc/ufw/sysctl.conf 
    SYSCALL . 09:26:10 . syscall=rename . pid's uid's comm=sed exe=/usr/bin/sed . key=UFW_rules 

<br>

The UFW_rules detects the changes to the /etc/ufw/sysctl.conf file.

<br>

### 8: Edit UFW firewall main configuration file

Command:

    echo "# THIS IS A COMMENT" >> /etc/default/ufw
    
Log:

    PROCTITLE . 09:26:14 . proctitle=/bin/bash /code/bsh/art/art_test.bsh /code/bsh/art/atomics/T1562.004.yaml y a 
    SYSCALL .   09:26:14 . syscall=openat . pid's uid's comm=art_test.bsh exe=/usr/bin/bash . key=UFW_config 

Command:

    sed -i 's/# THIS IS A COMMENT//g' /etc/default/ufw
    
Log:

    PROCTITLE . 09:26:16 . proctitle=sed -i s/# THIS IS A COMMENT//g /etc/default/ufw 
    SYSCALL . 09:26:16 . syscall=rename . pid's uid's comm=sed exe=/usr/bin/sed . key=UFW_config 

<br>

The UFW_config detects the changes to the /etc/default/ufw file.

<br>

### 9: Tail the UFW firewall log file

Command:

    tail /var/log/ufw.log
    
Log:

    PROCTITLE . 09:26:19 . proctitle=tail /var/log/ufw.log 
    SYSCALL .   09:26:19 . syscall=openat . pid's uid's comm=tail . key=UFW_log 

<br>

The UFW_config detects the read on the /var/log/ufw.log file.

<br>

### Conclusion

We only use watch rules, so they have very little effect on system performance, while still detecting all attempts to alter or stop the firewall.


<br>

### References:

- [MITRE ATT&CK - Impair Defenses: Disable or Modify System Firewall](https://attack.mitre.org/techniques/T1562/004/)
- [man auditd](https://manpages.ubuntu.com/manpages/xenial/en/man8/auditd.8.html)
- [man ufw](http://manpages.ubuntu.com/manpages/bionic/man8/ufw.8.html)

