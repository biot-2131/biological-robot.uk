---
layout: post
title: "So who have you been talking to?"
strapline: "Dumping DNS queries to a log file"
date: 2023-02-20 
published: true
permalink: so-who-have-you-been-talking-to
lastmod: 2023-02-20
changefreq: monthly
priority: 0.5
categories: tcpdump dns 
excerpt_separator: <!--excerpt-->
---

There are enumerable reasons why you would want to know who and when your computer has been communicating with. If we monitor and record the computers use of the Domain Name System (DNS) which is used to identify computers on the local network and the internet we would gain that knowledge. In this post we look at a way to use tcpdump to capture and log all of the DNS queries made by a computer.

<!--excerpt-->

<br>

## Requirements 

- Linux OS, systemd and tcpdump
- Human readable log file -> /var/log/
- Logrotate 9, 2mb gziped logs
- Capture all un-encrypted port 53 UDP
- Ignore DNS cache hits
- A background deamon, started on boot 
- Simple, not convoluted with loads of dependance's

<br>

To build this capability we use a Ubuntu 18.04 box running systemd and with tcpdump installed. 


To create a service that will start on boot we use systemd to create a service using a bash script and the setsid utility to turn the bash script into a daemon. 


To create the human readable log file we will use tcpdump in parse mode rather than capture mode (creating a pcap file) and redirect that output into the log. For example:

    .. localhost.51943 > localhost.domain: 37231+ [1au] A? ntp.ubuntu.com. (43)
    .. localhost.51943 > localhost.domain: 34160+ [1au] AAAA? ntp.ubuntu.com. (43)
    .. steelyglint.home.arpa.51707 > dramaticexit.home.arpa.domain: 64954+ [1au] A? ntp.ubuntu.com. (43)
    .. steelyglint.home.arpa.39832 > dramaticexit.home.arpa.domain: 36366+ [1au] AAAA? ntp.ubuntu.com. (43)
    .. dramaticexit.home.arpa.domain > steelyglint.home.arpa.51707: 64954 4/0/1 A 91.189.94.4, A 91.189.89.199, A 91.189.91.157, A 91.189.89.198 (107)
    .. localhost.domain > localhost.51943: 37231 4/0/1 A 91.189.94.4, A 91.189.89.199, A 91.189.91.157, A 91.189.89.198 (107)
    .. dramaticexit.home.arpa.domain > steelyglint.home.arpa.39832: 36366 2/0/1 AAAA 2001:67c:1560:8003::c7, AAAA 2001:67c:1560:8003::c8 (99)
    .. localhost.domain > localhost.51943: 34160 2/0/1 AAAA 2001:67c:1560:8003::c7, AAAA 2001:67c:1560:8003::c8 (99)

<br>

## dns_traffic.bsh bash script

    :~# nano /sbin/dns_traffic.bsh

        #!/bin/bash

        # if log directory does not exist, create it.
        if [ ! -d /var/log/dns_traffic ]
        then 
            mkdir /var/log/dns_traffic 
        fi

        # if log file does not exist, create it.
        if [ ! -e /var/log/dns_traffic/dns_traffic.log ]
        then 
            touch /var/log/dns_traffic/dns_traffic.log  
            chmod 0600 /var/log/dns_traffic/dns_traffic.log 
        fi

        # run tcpdump as a deamon
        setsid tcpdump -i any udp port 53 2>&1 >> /var/log/dns_traffic/dns_traffic.log

    :~# chmod +x /sbin/dns_traffic.bsh

<br>

## systemd service 

We build a systemd service which will start whenever the system boots.

    :~# nano /etc/systemd/system/dns_traffic.service
        [Unit]
        Description=tcpdump DNS traffic

        [Service]
        Type=simple
        ExecStart=/sbin/dns_traffic.bsh

        [Install]
        WantedBy=multi-user.target

    :~# chmod 644 /etc/systemd/system/dns_traffic.service

    :~# systemctl enable dns_traffic.service 
    :~# systemctl start dns_traffic.service
    :~# systemctl status dns_traffic.service

    :~# ps aux |grep "tcpdump"
        tcpdump     2289  ... tcpdump -i any udp port 53

    :~# ps aux |grep "traffic"
        root        2275  ... /bin/bash /sbin/dns_traffic.bsh


<br>

## Test functionality

To confirm that we are capturing all the DNS requests we will lookup the top 
1 million sites on the internet according to Alexa traffic ranking.

Download and unzip the Alexa rankings.

    :~# wget -v http://s3.amazonaws.com/alexa-static/top-1m.csv.zip
    :~# unzip top-1m.csv.zip 

Bash command to lookup the first one thousand sites 

    :~# i=1; for L in $(sed -n '1,1000p' top-1m.csv); do \
        D=$(echo $L |cut -d"," -f2); \
        echo "\$i=$i \$D=$D"; \
        dig A $D; \
        ((i = i + 1)); \
        done

    :~# tail -f /var/log/dns_traffic/dns_traffic.log 
    :~# ls -lah /var/log/dns_traffic/dns_traffic.log 
        -rw------- 1 root root 460K ... /var/log/dns_traffic/dns_traffic.log

Bash command to check that the DNS request was logged. Note if the name was
unresolvable their will be no entry in the log.

    :~# echo "" > not_found.txt; i=1; \
        for L in $(sed -n '1,1000p' top-1m.csv); do \
        D=$(echo $L |cut -d"," -f2); \
        if ! grep $D -q /var/log/dns_traffic/dns_traffic.log; then \
            echo "$i,$D -> not found!"; \
            echo "$i,$D" >> not_found.txt; \
        fi; \
        ((i = i + 1)); \
        done

<br>

## Logrotate

To prevent the log file from becoming to large and unwieldy we setup the 
following logrotate configuration.

    :~# nano /etc/logrotate.d/dns_traffic.conf 
        /var/log/dns_traffic/* {
            daily
            create 0600 root root
            rotate 9
            size=2M
            compress 
            delaycompress
        }

* daily means that the tool will attempt to rotate the logs on a daily basis. 
* rotate 9 indicates that only nine rotated logs should be kept. 
* size=2M sets the minimum size for the rotation to take place. 
* compress and delaycompress are used to gzip the rotated logs.

<br>

### dry-run logrotate

Let’s execute a dry-run to see what logrotate would do if it was actually 
executed now. Use the -d option followed by the configuration file. 

    :~# logrotate -d /etc/logrotate.d/dns_traffic.conf 

<br>

### Manually run logrotate

When you have a log file that is lager that 2mb, you can run logrotate manually.


    :~# logrotate /etc/logrotate.d/dns_traffic.conf 

<br>
<br>

## References

### configuration files

- [dns_traffic.bsh](/assets/text/dns_traffic.bsh)
- [dns_traffic.conf](/assets/text/dns_traffic.conf)
- [dns_traffic.service](/assets/text/dns_traffic.service)

<br>

### man pages

- [tcpdump & libpcap](https://www.tcpdump.org)
- [man tcpdump - dump traffic on a network](/assets/text/man_tcpdump.txt)
- [man setsid - run a program in a new session](/assets/text/man_setsid.txt)
- [systemd, init - systemd system and service manager](/assets/text/man_systemd.txt)
- [man dig - DNS lookup utility](/assets/text/man_dig.txt)

<br>

### web pages

- [Wiki Domain Name System](https://en.wikipedia.org/wiki/Domain_Name_System)
- [How to Setup and Manage Log Rotation Using Logrotate in Linux](https://www.tecmint.com/install-logrotate-to-manage-log-rotation-in-linux/)
- [Alexa - Top sites](https://www.alexa.com/topsites)

<br>
<br>

