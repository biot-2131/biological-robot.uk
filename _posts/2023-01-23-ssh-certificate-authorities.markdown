---
layout: post
title: "SSH Certificate Authorities"
strapline: '"Better" Key Management'
date: 2023-01-23 
published: true
permalink: ssh-certificate-authorities
lastmod: 2023-01-23
changefreq: monthly
priority: 0.5
categories: research howto ssh
excerpt_separator: <!--excerpt-->
---

Without good key management, no cryptographic system is secure. Full knowledge of the Who, What, When, Where, Why and How of your keys is imperative for good security. Using an SSH Certificate Authority (CA) will vastly simplify and improve the efficacy of SSH infrastructure. You will no longer have default usernames and passwords (root:toor). You will no longer have to copy everyone's public key into every server while trying to remember whose key is whos. You will no longer be tempted to create "One key to rule them ALL" and give that key to everyone who needs it. Your key management process will be reduced to installing a CA's public key to each system and signing users public keys.  

<!--excerpt-->

<br>

#### SSH Certificate Authorities

An SSH CA is simply a special set of public/private key pairs that are used to cryptographically sign user or hosts public keys creating certificates that can be used to authenticate users to hosts, or hosts to users. This mutual authentication scheme can be enforced on both user and hosts systems to provide a greater level of assurance, "to both parties" that they are talking to who they think they are talking to.

<br>

#### SSH Certificates 

Certificates: user and host. consist of a public key, some identity information, one or more principal (user or host) names and a set of options that are then signed by a Certification Authority (CA) key. These signing options allow for a finer degree of control when granting access. Some of the options are:

1. Certificates can have start and end times, meaning that you can issue in advance and automatically expire when required.
2. Source IP address restrictions, restricting the source address or CIDR allows for access segmentation within larger address ranges.
3. Force command, forcing the execution of command instead of initiating an interactive shell (scripting, automation tasks). 
4. Permit or deny port/agent forwarding, permitting/preventing lateral movement within the network.

<br>

#### SSH Key Revocation

Any key management process must have a key revocation mechanism. SSH uses Key Revocation Lists (KRLs) on each server. While the management of this list is a manual process it is effective at forbidding to use or revoked keys and logging every attempt.

<br>

#### Zero Trust 

While SSH CA's do not meet all the tenets of a Zero Trust - Access to Resources policy needs, they do meet the core requirements:

- Provision for mutual user and hosts authentication
- Access expiry and revocation
- Application functionality restrictions
- Network location restrictions
- Logging; enough information to build behavioural and environmental attributes for each user.
- Auditable; the configuration is auditable.

<br>

### SETUP

To test all of this theory I am going to build and configure four Ubuntu docker containers; three clients alice@heavymessing, bob@justtesting and chloe@tacticalgrace and one server user@lastingdamage. I will create an SSH Certificate Authority, distributes the keys and configure all of the clients and the server, to ONLY trust CA-signed certificates.

<p align="center" width="100%">
    <img width="49%" src="/assets/images/a-setup.png"> 
</p>

<br>

#### Root Vs. User

I am using a common user account on the server called "user", which implies that all users know the user's password to enable them to gain sudo privileges.  However, you could just as easily use the root account, immediately granting root privileges and bypass the need for any password sharing. The choice is yours. 

<br>

Create user keys (for each user@host)

    $ ssh-keygen -f /home/user/.ssh/id_rsa 

Create user & host CA certificates (root@lastingdamage)

    $ mkdir /root/ca_keys
    $ ssh-keygen -f /root/ca_keys/ca_user_key
    $ ssh-keygen -f /root/ca_keys/ca_host_key

Signing user's public keys (root@lastingdamage, for each user@host)

    $ ssh-keygen -s /root/ca_keys/ca_user_key -I user@host -V +30m \
      -O source-address=172.18.0.0/24 \
      -n user /root/public_keys/user-host.pub
    
Creates a /root/public_keys/user@host-cert.pub file i.e. a "CA signed" users public key that is valid for 30 minutes and its usage is restricted to 172.18.0.0/24 subnet.

NOTE: The -V validity interval option "IS" server timezone sensitive!

    $ PUBLIC=$(ssh-keygen -L -f /root/public_keys/user@host-cert.pub \
      |grep "Public" |sed -e 's/:/ /g' |sed 's/^[ \t]*//' |sed 's/  / /g' \
      |cut -d" " -f1,5).
      
grepping the public keys SHA256 hash

    $ SIGNING=$(ssh-keygen -L -f /root/public_keys/user@host-cert.pub \
      |grep "Signing" |sed -e 's/:/ /g' |sed 's/^[ \t]*//' |sed 's/  / /g' \
      |sed -e 's/\(using rsa-sha2-512\)//g' |cut -d" " -f1,5)
      
grepping the signing keys SHA256 hash

    $ echo "user@host $PUBLIC $SIGNING" >> /root/public_keys/public_key_index.txt 
   
The public_key_index.txt file contains a list of all the sign public keys.

<p align="center" width="100%">
    <img width="49%" src="/assets/images/07-public_key_index.png"> 
</p>

Server (root@lastingdamage)

    $ ssh-keygen -s /root/ca_keys/ca_host_key -I lastingdamage -h \
      -n lastingdamage -V +1d /etc/ssh/ssh_host_rsa_key.pub 
      
Sign servers public key creating /etc/ssh/ssh_host_rsa_key-cert.pub key that is valid for one day.

NOTE: The -V validity interval option "IS" server timezone sensitive!

<p align="center" width="100%">
    <img width="49%" src="/assets/images/d-cert.pub_ca.png"> 
</p>

Copy CA's public key 

    $ cp -fv /root/ca_keys/ca_user_key.pub /etc/ssh/

configure server /etc/ssh/sshd_config: 

    PermitRootLogin no
    PasswordAuthentication no
    PubkeyAuthentication yes
    LogLevel VERBOSE
    TrustedUserCAKeys /etc/ssh/ca_user_key.pub
    HostCertificate /etc/ssh/ssh_host_rsa_key-cert.pub
    RevokedKeys /etc/ssh/revoked_keys

LogLevel VERBOSE must be used because LogLevel INFO does not log the certificate ID's or key SHA256 hashes for login attempts that are made with expired certificates.

    $ touch /etc/ssh/revoked_keys && chmod 644 /etc/ssh/revoked_keys 

The revoked_keys file MUST exist and be readable (even if empty) otherwise the server will not accept ANY connections.

Copy signed certificates to ~/.ssh & update known_hosts (for each user@host)

    $ cp -fv /root/public_keys/user@host-cert.pub /home/user/.ssh/id_rsa-cert.pub
    $ printf "@cert-authority * " | cat - /root/ca_keys/ca_host_key.pub \
      > /home/user/.ssh/known_hosts 

The user's known hosts file now contains the CA's host's public key, meaning that the ssh client will allow connections to any server presenting this public key. 

<br>

### TESTS   


To gain a level of assurance that the required functionality has been achieved, conducted the following tests. 

<br>1\. User Identity

<div style="padding-left: 15px;">
Alice & Bob can login to user@lastingdamage without having someone copy their public keys to .ssh/authorized_keys first (ssh-copy-id) because they are using their signed public keys [name]-cert.pub. 
</div>

<br>2\. Server Identity

<div style="padding-left: 15px;">
Alice & Bob do NOT get prompted with an "unknown host" warning message because the ca_host_key.pub is in their .ssh/known_hosts file
</div>

<p align="center" width="100%">
    <img width="49%" src="/assets/images/02-alice_login.png"> 
    <img width="49%" src="/assets/images/02-bob_login.png"> 
</p>

3\. Identity - logging

<div style="padding-left: 15px;">
user@lastingdamage /var/log/auth.log should log individual certificate ID's.. tail -f /var/log/auth.log
</div>

<p align="center" width="100%">
    <img width="49%" src="/assets/images/03-alice_login_log.png"> 
    <img width="49%" src="/assets/images/03-bob_login_log.png"> 
</p>

<br>4\. Identity - revocation

<div style="padding-left: 15px;">
Server (root@lastingdamage)
</div>

    $ ssh-keygen -kf /root/ca_keys/revoked_keys -z $COUNTER bob-justtesting.pub 
    $ cp -fv /root/ca_keys/revoked_keys /etc/ssh/
    $ chmod 644 /etc/ssh/revoked_keys

<div style="padding-left: 15px;">
Bob should NOT be able to login to user@lastingdamage after his certificate is revoked.
</div>

<p align="center" width="100%">
    <img width="49%" src="/assets/images/04-bob_revoked_login.png"> 
    <img width="49%" src="/assets/images/04-bob_revoked_log.png"> 
</p>

<br>5\. Identity - expire

<div style="padding-left: 15px;">
Alice should NOT be able to login to user@lastingdamage after her certificate expires. 
</div>

<p align="center" width="100%">
    <img width="49%" src="/assets/images/05-alice_expire_login.png"> 
    <img width="49%" src="/assets/images/05-alice_expire_log.png"> 
</p>

NOTE: The Valid: from to values "ARE" server timezone sensitive!

<br>6\. Network location

<div style="padding-left: 15px;">
Chloe can't ssh to user@lastingdamage from outside 172.18.0.0/24 CIDR, even with valid keys.
</div>

<p align="center" width="100%">
    <img width="49%" src="/assets/images/06-chloe_location_login.png"> 
    <img width="49%" src="/assets/images/06-chloe_location_log.png"> 
</p>

<br>7\. Identity - registry

<div style="padding-left: 15px;">
A public_keys/public_key_index.txt file is created with an index of ID's and their PUBLIC keys which have been signed -> used for identifying users when greping auth.log
</div>

<p align="center" width="100%">
    <img width="49%" src="/assets/images/07-public_key_index.png"> 
</p>

<br>

### Conclusion

SSH key management is very simple:
- All servers are configured to use CA keys and certificates
- The user provides their public key, CA signs the key and returns the resulting user certificate and the CA's public key.
- The user configures their system and use the CA's signed certificate.
- The user keys either expire or are revoked as required.
- SSH auth.log records all connection activity.

You are making "One key to rule them ALL"  but it is a Certificate Authority key that never needs to be on an online system, it could/should reside on a USB key and only plugged in when needed.
 
<br>

### References:

MAN pages

- [man ssh-copy-id](/assets/text/man_ssh-copy-id.txt)
- [man ssh-keygen](/assets/text/man_ssh-keygen.txt)
- [man ssh](/assets/text/man_ssh.txt)
- [man sshd](/assets/text/man_sshd.txt)

NIST Special Publication 800-57 Recommendation for Key Management - Part 1: General (Revision 3)
<br>Elaine Barker, William Barker, William Burr, William Polk, and Miles Smid 
[https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-57p1r3.pdf](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-57p1r3.pdf)

NIST Special Publication 800-207 - Zero Trust Architecture 
<br>Scott Rose Oliver Borchert Stu Mitchell Sean Connelly 
[https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-207.pdf)

