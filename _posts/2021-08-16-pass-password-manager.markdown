---
layout: post
title: "pass Password Manager"
strapline: "Shared passwords"
date: 2021-08-16 
published: true
lastmod: 2021-08-16
changefreq: monthly
priority: 0.5
categories: research howto pass
excerpt_separator: <!--excerpt-->
---

Love them or hate them, shared passwords are a fact of life. Default user accounts, database users, API keys etc. pass the "standard Unix password manager" stores each password within a directory structure of gpg encrypted files. Because gpg is used, multiple gpg id's can be used to encrypt/decrypt the password files and due to the simple directory and file structure, git can be used to track changes and share the password repository with others.

<!--excerpt-->

<br>

### SETUP

To set up and test out sharing passwords between two people, I will build and configure three Ubuntu docker containers; two clients alice@heavymessing, bob@justtesting to act as user machines and server git@lastingdamage to be the Git repository machine. 

<p align="center" width="100%">
    <img width="49%" src="/assets/images/pass-password-manager/a-setup.png"> 
</p>

<br>

Each host requires these packages:

    openssh-client - secure shell (SSH) client, for access to remote machines
    gnupg - GNU privacy guard - a free PGP replacement
    pass - lightweight directory-based password manager
    git - fast, scalable, distributed revision control system

<br>

### gpg
Each user will need to generate their own keys, export their public key and import and sign their team members public key. 

    gpg --full-generate-key
    gpg --export --armor $USER@$HOST > .gnupg/$USER@$HOST_public.asc
    gpg --import /root/public_keys/bob@justtesting_public.asc
    gpg --edit-key bob@justtesting -> lsign

Note: Permission denied error

    gpg: agent_genkey failed: Permission denied
    Key generation failed: Permission denied

is fixed by 

    ls -la $(tty)
    sudo chown <USERNAME> $(tty)

<br>

### pass
Each user will need to setup pass. 

    pass init $(echo $USER)@$(echo $HOST)
    pass generate $(echo $HOST)/$(echo $USER) 15
    pass generate gmail/$USER 15
    pass

<p align="center" width="100%">
    <img width="49%" src="assets/images/pass-password-manager/01_password_store_alice.png"> 
    <img width="49%" src="assets/images/pass-password-manager/01_password_store_bob.png"> 
</p>

<br>

### Git
Each user will need to setup git personal password repo. 

    cd .password-store
    git init
    git add .
    git commit -m "first commit"
    git remote add origin git@lastingdamage:password-store-$USER.git

Alice sets up a shared password repo. 

    cd /home/alice/.password-store
    git push --set-upstream origin master

    echo "shared" > .gitignore 
    git clone git@lastingdamage:password-store-shared.git shared
    pass init -p shared alice@heavymessing
    echo "bob@justtesting" >> /home/alice/.password-store/shared/.gpg-id
    pass init -p shared $(cat /home/alice/.password-store/shared/.gpg-id)
    pass generate --no-symbols shared/password 15
    pass

    cd /home/alice/.password-store/shared
    git add .
    git commit -m "added shared/password"
    git push --set-upstream origin master

Bob clones the shared password repo.

    cd /home/alice/.password-store
    git push --set-upstream origin master

    echo "shared" > .gitignore 
    git clone git@lastingdamage:password-store-shared.git shared
    pass

<p align="center" width="100%">
    <img width="49%" src="assets/images/pass-password-manager/02_git_repos_gpg-ids_alice.png"> 
    <img width="49%" src="assets/images/pass-password-manager/02_git_repos_gpg-ids_bob.png"> 
</p>

<br>

### TESTS   

To gain a level of assurance that the required functionality has been achieved, conducted the following tests. 

<br>1\. Alice & Bob have their individual and shared password stores

<p align="center" width="100%">
    <img width="49%" src="assets/images/pass-password-manager/02_git_repos_alice.png"> 
    <img width="49%" src="assets/images/pass-password-manager/02_git_repos_bob.png"> 
</p>

<br>2\. Alice changes the shared password and push's the git repo and Bob pulls

<p align="center" width="100%">
    <img width="49%" src="assets/images/pass-password-manager/03_change_password_alice_01.png"> 
    <img width="49%" src="assets/images/pass-password-manager/03_change_password_bob.png"> 
</p>

<br>4\. Alice removes Bob's id and changes the shared password and push's the git repo. Bob pulls the shared repo, but is unable to decrypt the password.

<p align="center" width="100%">
    <img width="49%" src="assets/images/pass-password-manager/04_remove_bob_alice.png"> 
    <img width="49%" src="assets/images/pass-password-manager/04_remove_bob_bob.png"> 
</p>

<br>

### Conclusion

pass is very easy to use, git and gpg are straightforward, making sharing password simple and secure. 

<br>

### References:

- [pass - the standard unix password manager](https://www.passwordstore.org)
- [password-store](https://git.zx2c4.com/password-store/about/)
- [pass](/assets/text/pass.txt)
- [man pass](/assets/text/man_pass.txt)
- [man gpg](/assets/text/man_gpg.txt)


