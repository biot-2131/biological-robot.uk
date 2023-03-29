---
layout: post
title: "Systemd timers"
strapline: "T1053.006 - MITRE ATT&CK"
date: 2021-11-26 
published: true
permalink: systemd-timers
lastmod: 2021-11-26
changefreq: monthly
priority: 0.5
categories: ATT&CK Atomic systemd auditd
excerpt_separator: <!--excerpt-->
---

Red Hat started the development of systemd in 2010 and by 2015 most Linux distributions had adopted it as their default system and service manager. One of the many capabilities of systemd is its ability to use timers. Which are similar to the older cron job scheduler but with extended capabilities.

Timers can be run by system or a standard user. They can be either Monotonic meaning that they run after a specified event such as boot or they can be realtime i.e. run at specified times, just like cron. They can also be persistent with files describing their functionality or they can be transient without definition files.

In this blog post we investigate four different ways that timers can be created and executed, either by the system or a standard user. We also look for ways in which the creation and execution of these timers could be detected and reported to an Endpoint Detection and Response system and map that activity back to the MITE Attack frameworks - Scheduled Task/Job: Systemd Timers - T1053.006 definition.

<!--excerpt-->

<br>

#### [From MITE Attack - Scheduled Task/Job: Systemd Timers T1053.006](https://attack.mitre.org/techniques/T1053/006/)

> Adversaries may abuse systemd timers to perform task scheduling for initial or recurring execution of malicious code. Systemd timers are unit files with file extension .timer that control services. Timers can be set to run on a calendar event or after a time span relative to a starting point. They can be used as an alternative to Cron in Linux environments. Systemd timers may be activated remotely via the systemctl command line utility, which operates over SSH.

> Each .timer file must have a corresponding .service file with the same name, e.g., example.timer and example.service. .service files are Systemd Service unit files that are managed by the systemd system and service manager. Privileged timers are written to /etc/systemd/system/ and /usr/lib/systemd/system while user level are written to ~/.config/systemd/user/.

> An adversary may use systemd timers to execute malicious code at system startup or on a scheduled basis for persistence. Timers installed using privileged paths may be used to maintain root level persistence. Adversaries may also install user level timers to achieve user level persistence.

<br>

## Techniques 

<br>

For these techniques, we will only use realtime timers because we want to run services every minute to see their effects, rather than having to wait or trigger specific events. We will also use a Ubuntu 18.04.1 LTS VM. All of these techniques "should" work on any Linux distribution that has implemented systemd as long as you adjust executable paths to be absolute.

<br>

### 1. System, Realtime & Persistant

    :~# nano /sbin/badstuff-SRP.sh
        #!/bin/sh
        echo -e "\n$(date) - $(whoami)" >> /tmp/badstuff-SRP.log

    :~# chmod +x /sbin/badstuff-SRP.sh

    :~# nano /etc/systemd/system/badstuff-SRP.service
        [Unit]
        Description=Demonstrate a System, Realtime & Persistant systemd service
        Wants=badstuff-SRP.timer

        [Service]
        Type=oneshot
        ExecStart=/sbin/badstuff-SRP.sh
        ExecStart=/bin/sh -c '/bin/pwd >> /tmp/badstuff-SRP.log'

        [Install]
        WantedBy=multi-user.target

    :~# nano /etc/systemd/system/badstuff-SRP.timer
        [Unit]
        Description=Demonstrate a System, Realtime & Persistant systemd timer
        Requires=badstuff-SRP.service

        [Timer]
        Unit=badstuff-SRP.service
        OnCalendar=*-*-* *:*:00

        [Install]
        WantedBy=timers.target

    :~# systemctl start badstuff-SRP.timer
    :~# systemctl status badstuff-SRP.timer
    :~# systemctl status badstuff-SRP.service

    :~# journalctl -S today -f -u badstuff-SRP.timer
        ... Started Demonstrate a System, Realtime & Persistant systemd timer.
        ... Starting Demonstrate a System, Realtime & Persistant systemd service.
        ... SERVICE_START pid=1 uid=0 . msg='unit=badstuff-SRP comm="systemd" exe="/lib/systemd/systemd"
        ... SERVICE_STOP pid=1 uid=0 . msg='unit=badstuff-SRP comm="systemd" exe="/lib/systemd/systemd"
        ... Started Demonstrate a System, Realtime & Persistant systemd service.

    :~# cat /tmp/badstuff-SRP.log
        Wed Nov 24 14:36:02 UTC 2021 - root
        /

    :~# systemctl stop badstuff-SRP.timer
    :~# systemctl stop badstuff-SRP.service
    :~# systemctl daemon-reload

<br>

### 2. System, Realtime & Transient

    :~# systemd-run --no-ask-password \
        --unit=System-Realtime-Transient-systemd-service \
        --on-calendar '*:0/1' \
        /bin/sh -c 'echo "$(date) $(whoami)" >> /tmp/badstuff-SRT.log'

        Running timer as unit: System-Realtime-Transient-systemd-service.timer
        Will run service as unit: System-Realtime-Transient-systemd-service.service

    :~# find /run/systemd/transient/
        /run/systemd/transient/System-Realtime-Transient-systemd-service.service
        /run/systemd/transient/System-Realtime-Transient-systemd-service.timer

    :~# systemctl status System-Realtime-Transient-systemd-service.timer
    :~# systemctl status System-Realtime-Transient-systemd-service.service

    :~# journalctl -S today -f -u System-Realtime-Transient-systemd-service
        -- Logs begin at Tue 2021-11-02 10:50:21 UTC. --
        ... Started /bin/sh -c echo "$(date) $(whoami)" >> /tmp/badstuff-SRT.log.

    :~# cat /tmp/badstuff-SRT.log
        Wed Nov 24 14:55:09 UTC 2021 root

    :~# systemctl stop System-Realtime-Transient-systemd-service.timer
    :~# systemctl daemon-reload

<br>

### 3. User, Realtime & Persistant

    :~$ mkdir -p $HOME/bin $HOME/.config/systemd/user  

    :~$ nano $HOME/bin/badstuff-URP.sh
        #!/bin/sh
        echo "\n$(date) - $(whoami)" >> /tmp/badstuff-URP.log  

    :~$ chmod +x $HOME/bin/badstuff-URP.sh

    :~$ nano $HOME/.config/systemd/user/badstuff-URP.service
        [Unit]
        Description=Demonstrate a User, Realtime & Persistant systemd service
        Wants=badstuff-URP.timer

        [Service]
        Type=oneshot
        ExecStart=/home/ubuntu/bin/badstuff-URP.sh
        ExecStart=/bin/sh -c '/bin/pwd >> /tmp/badstuff-URP.log'

        [Install]
        WantedBy=multi-user.target

    :~$ nano $HOME/.config/systemd/user/badstuff-URP.timer
        [Unit]
        Description=Demonstrate a User, Realtime & Persistant systemd timer
        Requires=badstuff-URP.service

        [Timer]
        Unit=badstuff-URP.service
        OnCalendar=*-*-* *:*:00

        [Install]
        WantedBy=timers.target

    :~$ systemctl --user start badstuff-URP.timer
    :~$ systemctl --user status badstuff-URP.timer
    :~$ systemctl --user status badstuff-URP.service
    :~$ systemctl --user --all list-timers 

    :~$ journalctl --user -S today -f
        ... Starting Demonstrate a User, Realtime & Persistant systemd service...
        ... Started Demonstrate a User, Realtime & Persistant systemd service.

    :~$ cat /tmp/badstuff-URP.log 
        Wed Nov 24 16:57:54 UTC 2021 - ubuntu
        /home/ubuntu

    :~$ systemctl --user stop badstuff-URP.timer
    :~$ systemctl --user stop badstuff-URP.service
    :~$ systemctl --user daemon-reload

<br>

### 4. User, Realtime & Transient

    :~$ systemd-run --user --no-ask-password \
        --unit=User-Realtime-Transient-systemd-service \
        --on-calendar '*:0/1' \
        /bin/sh -c 'echo "$(date) $(whoami)" >> /tmp/badstuff-URT.log'

        Running timer as unit: User-Realtime-Transient-systemd-service.timer
        Will run service as unit: User-Realtime-Transient-systemd-service.service

    :~$ systemctl --user status User-Realtime-Transient-systemd-service.timer
    :~$ systemctl --user status User-Realtime-Transient-systemd-service.service
    :~$ systemctl --user --all list-timers 

    :~$ find /run/user/
        /run/user/1000/systemd/transient/User-Realtime-Transient-systemd-service.service
        /run/user/1000/systemd/transient/User-Realtime-Transient-systemd-service.timer

    :~$ journalctl --user -S today -f
        ... Started /bin/sh -c echo "$(date) $(whoami)" >> /tmp/badstuff-URT.log.

    :~$ cat /tmp/badstuff-URT.log
        Wed Nov 24 17:09:54 UTC 2021 ubuntu

    :~$ systemctl --user stop User-Realtime-Transient-systemd-service.timer
    :~$ systemctl --user daemon-reload

<br>

## Detection

### systemd journal

The starting, stopping and reloading of timers is recorded in the systemd journal

    :~# journalctl -S today -f

Note that the systemctl --all list-timers does not show the user created timers
you have to use systemctl --user --all list-timers for each user.

<br>

### System utilities 

The three utilities that are used to start, stop, reload and monitor systemd 
services and timers can be monitored by auditd for their execution activity.

    :~$ auditctl -w /bin/journalctl -p x -k systemd
    :~$ auditctl -w /bin/systemctl -p x -k systemd
    :~$ auditctl -w /sbin/augenrules -p x -k systemd
    :~$ auditctl -w /usr/bin/systemd-run -p x -k systemd

<br>

### Persistant timer directories

Persistant timers require the *.service and *.timer files are created either 
/etc/systemd/system/ for system and /home/\[username\]/.config/systemd/user/ 
created timers so auditd can be used to monitor those directories for changes.

    :~$ auditctl -w /etc/systemd/system/ -k systemd
    :~$ auditctl -w /home/[username]/.config/systemd/user/ -k systemd

<br>

### Transient timer directories

When Transient timers are created *.service and *.timer files are created 
automatically by the API in either /run/systemd/transient/ for system and 
/run/user/\[uid\]/systemd/transient/ for user timers so auditd can be used to 
monitor those directories for changes.

    :~$ auditctl -w /run/systemd/transient/ -k systemd
    :~$ auditctl -w /run/user/1000/systemd/transient/ -k systemd

<br>

### Building auditd rules

    nano /etc/audit/rules.d/T1053.006.rules

        # auditd rules MITE Attack - Scheduled Task/Job: Systemd Timers T1053.006
        
        -w /bin/journalctl -p x -k systemd
        -w /bin/systemctl -p x -k systemd
        -w /sbin/augenrules -p x -k systemd
        -w /usr/bin/systemd-run -p x -k systemd

        -w /etc/systemd/system/ -p w -k systemd
        -w /home/[username]/.config/systemd/user/ -p w -k systemd
        -w /run/systemd/transient/ -p w -k systemd
        -w /run/user/1000/systemd/transient/ -p w -k systemd

    augenrules --check
    augenrules --load
    auditctl -l

<br>

### Finding the goodies

    :~# ausearch -ts today -i -k systemd
    :~# ausearch -ts today -i -k systemd |grep proctitle=.*
    :~# tail -f /var/log/audit/audit.log |grep key=\"systemd\"

<br>

## References

### man pages

- [man systemd, init - systemd system and service manager](/assets/text/man_systemd.txt)
- [man systemd-run - Run programs in transient scope](/assets/text/man_systemd-run.txt)
- [man systemctl - Control the systemd system and service manager](/assets/text/man_systemctl.txt)
- [man journalctl - Query the systemd journal](/assets/text/man_journalctl.txt)
- [man auditd - The Linux Audit daemon](/assets/text/man_auditd.txt)
- [man auditctl - a utility to assist controlling the kernel's audit system](/assets/text/man_auditctl.txt)

<br>

### web pages

- [MITE Attack - Scheduled Task/Job: Systemd Timers T1053.006](https://attack.mitre.org/techniques/T1053/006/)
- [Wikipedia systemd](https://en.wikipedia.org/wiki/Systemd)
- [How to schedule tasks with systemd timers in Linux](https://linuxconfig.org/how-to-schedule-tasks-with-systemd-timers-in-linux)
- [Use systemd timers instead of cronjobs](https://opensource.com/article/20/7/systemd-timers)

<br>
<br>
