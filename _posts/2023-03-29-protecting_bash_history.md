---
layout: post
title: "Protecting Bash History"
strapline: "MITRE ATT&CK, T1562.003 - Impair Defenses: HISTCONTROL"
date: 2023-03-29 
published: true
permalink: protecting-bash-history
lastmod: 2023-03-29
changefreq: monthly
priority: 0.5
excerpt_separator: <!--excerpt-->
---

Adversaries use a variety of tactics to evade detection and maintain their presence on a compromised system. One such technique is to impair command history logging, making it difficult for security teams to detect and analyze the actions taken by the attacker.

In this blog post, we will be discussing the MITRE ATT&CK technique T1562.003 - Impair Defenses: HISTCONTROL, which involves adversaries manipulating the HISTCONTROL environment variable to prevent the logging of certain commands in the user's command history.

We will provide examples of how adversaries can exploit this technique and also discuss auditd rules that can be used to detect such activity. Finally, we will suggest some best practices to mitigate the risk of this technique being used against your organization.

<!--excerpt-->

![alt text](/assets/images/t1562.003-prevention.png)

> MITRE ATT&CK, technique T1562.003 - Impair Defenses: HISTCONTROL
>
>    Adversaries may impair command history logging to hide commands they run on a compromised system. Various command interpreters keep track of the commands users type in their terminal so that users can retrace what they've done.
> 
>    On Linux and macOS, command history is tracked in a file pointed to by the environment variable HISTFILE. When a user logs off a system, this information is flushed to a file in the user's home directory called /.bash_history. The HISTCONTROL environment variable keeps track of what should be saved by the history command and eventually into the /.bash_history file when a user logs out. HISTCONTROL does not exist by default on macOS, but can be set by the user and will be respected.
>
>    Adversaries may clear the history environment variable (unset HISTFILE) or set the command history size to zero (export HISTFILESIZE=0) to prevent logging of commands. Additionally, HISTCONTROL can be configured to ignore commands that start with a space by simply setting it to "ignorespace". HISTCONTROL can also be set to ignore duplicate commands by setting it to "ignoredups". In some Linux systems, this is set by default to "ignoreboth" which covers both of the previous examples. This means that “ ls” will not be saved, but “ls” would be saved by history. Adversaries can abuse this to operate without leaving traces by simply prepending a space to all of their terminal commands.

<br>

## Bash history environment variables

#### HISTFILE environment variable

The HISTFILE environment variable is used to specify the path and file name for the command history file. The valid settings for the HISTFILE variable include:

- An absolute or relative path to a file where the command history should be saved
- The value "dev/null", which will cause the command history to be discarded
- An empty value, which will prevent command history from being saved to a file

Note: that if the HISTFILE variable is not set, the default location for the command history file is "~/.bash_history".

#### HISTCONTROL environment variable

The HISTCONTROL variable is used to control what is saved to the history file and what is not. It is used by the history command and eventually by the shell to decide which commands should be saved to the history file when a user logs out. By setting this variable, users can control the types of commands that are saved and can also prevent specific commands from being saved to the history file. This is useful for preventing sensitive information such as passwords from being saved to the history file.

- ignorespace: If this setting is enabled, any command that begins with a space character will not be saved in the history list.
- ignoredups: This setting tells the shell to ignore duplicate commands in the history list.
- ignoreboth: This setting combines the ignorespace and ignoredups settings.
- erasedups: This setting tells the shell to remove duplicates from the history list.
- erasespaces: This setting causes the shell to remove commands that begin with spaces from the history list.

#### HISTFILESIZE environment variable

The HISTFILESIZE variable determines the maximum size of the history file for the current shell session. The following are the valid settings for the HISTFILESIZE variable:

- A positive integer: Specifies the maximum number of lines that can be stored in the history file. Once the file reaches this limit, the oldest commands are deleted from the file as new commands are added.
- "unlimited": Specifies that there is no limit to the size of the history file. This can be risky as it can potentially fill up the disk space if left unchecked.
- "0": Specifies that no commands should be saved to the history file. This effectively disables command history logging for the current shell session.

#### HISTSIZE environment variable

HISTSIZE is an environment variable that determines the maximum number of commands that can be saved in the history list. When a user logs out, the history commands are saved to a history file (usually ~/.bash_history), and when the user logs in again, the previous history is reloaded from this file.

The default value for HISTSIZE is usually 500, meaning that the shell keeps track of the last 500 commands. However, this value can be changed by the user or by system administrators. If the value is set to zero, no history commands will be saved, and the user will not be able to recall any previous commands.

It's important to note that HISTSIZE only determines the maximum number of commands that can be saved in the history list. If the number of commands exceeds this limit, the oldest commands will be dropped to make room for the new ones. Therefore, if a user wants to keep a longer history, they should increase the value of HISTSIZE.

#### HISTIGNORE environment variable

The HISTIGNORE environment variable is used to specify patterns that should be ignored by the bash history function. The variable contains a list of colon-separated patterns, and any command that matches one of these patterns will not be added to the history list. This can be useful for excluding frequently typed or sensitive commands from the history.

<br>

## Techniques 

#### 1. Clear bash history
An attacker may clear the bash history cache and the history file as their last act before logging off to remove the record of their command line activities. 

In this test we use the $HISTFILE variable throughout to 1. confirms the $HISTFILE variable is set 2. echo "" into it 3..5 confirm the file is empty 6 clear the history cache 7. confirm the history cache is empty. This is when the attacker would logoff. 
``` bash
if ((${#HISTFILE[@]})); then echo $HISTFILE; fi
echo "" > $HISTFILE
if [ $(wc -c <$HISTFILE) -gt 1 ]; then echo "$HISTFILE is larger than 1k"; fi
ls -la $HISTFILE 
cat $HISTFILE

history -c 
if [ $(history |wc -l) -eq 1 ]; then echo "History cache cleared"; fi
```
<br>

#### 2. Setting the HISTCONTROL environment variable
An attacker may exploit the space before a command (e.g. " ls") or the duplicate command suppression feature in Bash history to prevent their commands from being recorded in the history file or to obscure the order of commands used. 

In this test we 1. sets $HISTCONTROL to ignoreboth 2. clears the history cache 3. executes ls -la with a space in-front of it 4. confirms that ls -la is not in the history cache 5. sets $HISTCONTROL to erasedups 6. clears the history cache 7..9 executes ls -la $HISTFILE 3 times 10. confirms that their is only one command in history
``` bash
if [ "$HISTCONTROL" != "ignoreboth" ]; then export HISTCONTROL="ignoreboth"; fi
history -c 
 ls -la $HISTFILE # " ls -la $HISTFILE"
if [ $(history |wc -l) -eq 1 ]; then echo "ls -la is not in history cache"; fi
# -> ls -la is not in history cache
if [ "$HISTCONTROL" != "erasedups" ]; then export HISTCONTROL="erasedups"; fi
history -c 
ls -la $HISTFILE
ls -la $HISTFILE
ls -la $HISTFILE
if [ $(history |wc -l) -eq 2 ]; then echo "Their is only one entry for ls -la $HISTFILE"; fi
# -> Their is only one entry for ls -la
```
<br>

#### 3. Setting the HISTFILESIZE environment variable
An Adversary may set the bash history files size environment variable (HISTFILESIZE) to zero to prevent the logging of commands to the history file after they log out of the system.

Note: we don't wish to log out, so we are just confirming the value of HISTFILESIZE. In this test we 1. echo HISTFILESIZE 2. set it to zero 3. confirm that HISTFILESIZE is set to zero.
``` bash
echo $HISTFILESIZE
export HISTFILESIZE=0
if [ $(echo $HISTFILESIZE) -eq 0 ]; then echo "\$HISTFILESIZE is zero"; fi
# -> $HISTFILESIZE is zero
```
<br>

#### 4. Setting the HISTFILE environment variable 
An Adversary may clear, unset or redirect the history environment variable HISTFILE to prevent logging of commands to the history file after they log out of the system.

Note: we don't wish to log out, so we are just confirming the value of HISTFILE. In this test we 1. echo HISTFILE 2. set it to /dev/null 3. confirm that HISTFILE is set to /dev/null.
``` bash
echo $HISTFILE
export HISTFILE="/dev/null"
if [ $(echo $HISTFILE) == "/dev/null" ]; then echo "\$HISTFILE is /dev/null"; fi
# -> $HISTFILE is /dev/null
```
<br>

#### 5. Setting the HISTIGNORE environment variable
An Adversary may take advantage of the HISTIGNORE environment variable either to ignore particular commands or all commands. 

In this test we 1. set HISTIGNORE to ignore ls, rm and ssh commands 2. clear this history cache 3..4 execute ls commands 5. confirm that the ls commands are not in the history cache 6. unset HISTIGNORE variable 7..17 same again, but ignoring ALL commands.
``` bash
if ((${#HISTIGNORE[@]})); then echo "\$HISTIGNORE = $HISTIGNORE"; else export HISTIGNORE='ls*:rm*:ssh*'; echo "\$HISTIGNORE = $HISTIGNORE"; fi
# -> $HISTIGNORE = ls*:rm*:ssh*
history -c 
ls -la $HISTFILE
ls -la ~/.bash_logout
if [ $(history |wc -l) -eq 1 ]; then echo "ls commands are not in history"; fi
# -> ls commands are not in history
unset HISTIGNORE

if ((${#HISTIGNORE[@]})); then echo "\$HISTIGNORE = $HISTIGNORE"; else export HISTIGNORE='*'; echo "\$HISTIGNORE = $HISTIGNORE"; fi
# -> $HISTIGNORE = *
history -c 
whoami
groups
if [ $(history |wc -l) -eq 0 ]; then echo "History cache is empty"; fi
# -> History cache is empty
unset HISTIGNORE
```
<br>

## Detection & logging

#### HISTTIMEFORMAT variable
The HISTTIMEFORMAT variable is an environment variable in Bash that controls the format of timestamps for command history. By default, Bash does not include timestamps for command history, but if HISTTIMEFORMAT is set, it will use that format to include a timestamp for each command in the history file.

The format of the timestamp is a string that can include special codes that represent various parts of the date and time, such as %Y for the year, %m for the month, %d for the day, %H for the hour, %M for the minute, and %S for the second. For example, setting HISTTIMEFORMAT to "%F %T " would add a timestamp in the format "YYYY-MM-DD HH:MM:SS" to each command in the history file.

The HISTTIMEFORMAT variable is particularly useful for auditing purposes, as it can help to determine when a particular command was executed. However, it should be noted that it is not foolproof, as an adversary with sufficient privileges can modify the history file or unset HISTTIMEFORMAT to hide their tracks.

``` bash
export HISTTIMEFORMAT="%F %T "
whoami
groups
history 
#    1  2023-03-28 10:01:04 export HISTTIMEFORMAT="%F %T "
#    2  2023-03-28 10:01:07 whoami
#    3  2023-03-28 10:01:09 groups
#    4  2023-03-28 10:01:13 history
```
<br>

#### Auditd logging 
To detect adversaries from tampering with command history logging, system administrators can implement certain auditd rules on Linux systems. Auditd is a powerful auditing tool that monitors and logs system activity, including user commands.

##### 1. Export rule
To detect the execution of the export command, use the following auditd rule:
``` bash
auditctl -a always,exit -F arch=b64 -S execve -F exe=/usr/bin/export -k setenv
``` 
This rule has the following short comings: 
- I will fire every time the export command is used 
- It doesn't record which environment variable it was used with 
- Users can by pass it by not using the export command i.e. HISTIGNORE='*'

##### 2. bash_history.sh rule
If you plan to implement the prevention method below. Monitor for changes to the /etc/profile.d/bash_history.sh, with the following auditd rule:
``` bash
auditctl -w /etc/profile.d/bash_history.sh -p wa -k T1562.003-bash-history
```

<br>

## Prevention 

#### bash_history.sh
The /etc/profile.d directory is a standard directory on Unix-like systems that contains shell scripts to be executed by the system-wide bash shell during login. The purpose of these scripts is to provide system-wide environment variables and settings for all users.

Create a shell script file named /etc/profile.d/bash_history.sh with the following:
``` bash
#!/bin/bash
# This script sets several environment variables related to the Bash history
# to readonly mode.
# HISTFILE: specifies the location where the history file is saved.
# HISTFILESIZE: sets the maximum number of commands that can be saved. 
# HISTCONTROL: erasespaces: log commands starting with spaces 
# HISTIGNORE: don't ignore any commands. 
# HISTTIMEFORMAT: the timestamp format 
# Note: timestamps are seconds since the epoch use date --date='@2147483647' to 
# convert seconds to date time stamp 

readonly HISTFILE="$HOME/.bash_history"
export HISTFILE

readonly HISTFILESIZE=2000
export HISTFILESIZE

readonly HISTCONTROL="erasespaces"
export HISTCONTROL

readonly HISTIGNORE=""
export HISTIGNORE

readonly HISTTIMEFORMAT="%F %T "
export HISTTIMEFORMAT
```

**Reboot...Login!**

``` bash
env |grep HIST
    HISTCONTROL=erasespaces
    HISTTIMEFORMAT=%F %T 
    HISTFILE=/home/biot/.bash_history
    HISTIGNORE=
    HISTFILESIZE=2000

echo $HISTFILE
export HISTFILE=/dev/null
-bash: HISTFILE: readonly variable

echo $HISTFILESIZE
export HISTFILESIZE=0
-bash: HISTFILESIZE: readonly variable

echo $HISTCONTROL
export HISTCONTROL=""
-bash: HISTCONTROL: readonly variable

echo $HISTIGNORE
export HISTIGNORE="*"
-bash: HISTIGNORE: readonly variable

echo $HISTTIMEFORMAT
export HISTTIMEFORMAT=""
-bash: HISTTIMEFORMAT: readonly variable
```
<br>

The bash history file will now have timestamps that are seconds since the epoch. You can grep them out or use this script to place the timestamps and commands on the same line.
``` bash
cat $HISTFILE |grep --invert-match '^#'

# or

nano read_history.sh
#!/bin/bash
# This script reads in a bash history file $HISTFILE with epoch timestamps on 
# every second line and outputs the timestamps and commands on the same line. 

while IFS= read -r line; do
    if $(echo $line |grep -q '^#')
    then
        ts="@$(echo $line |cut -d"#" -f2)"
        echo -n "$(date --date=$ts) "
    else 
        echo "$line"
    fi 
done < $HISTFILE

./read_history.sh
```
<br>

## Conclusion

Implementing the bash_history.sh script can be a useful step in improving the security of a Linux system by preventing users from modifying or tampering with the variables associated with the bash history. In addition, using the HISTTIMEFORMAT variable to add timestamps to the bash history can provide valuable information for auditing and forensic purposes. This can help track when specific commands were executed and by whom, which can aid in identifying any malicious or unauthorized activities on the system.

However, it is important to note that the variables are only set to readonly when a user logs in directly and not when they use "su" or "sudo su" to switch to another user. Therefore, it is important to ensure that all users are aware of the importance of maintaining the integrity of the bash history and dithered from using the switch user functionality.

Ten points if you got this far, hopefully you found something useful. :-) 
<br>

#### Links
- MITRE ATT&CK: [https://attack.mitre.org/](https://attack.mitre.org/)
- T1562.003 - Impair Defenses: HISTCONTROL: [https://attack.mitre.org/techniques/T1562/003/](https://attack.mitre.org/techniques/T1562/003/)
- The official GNU Bash manual: [https://www.gnu.org/software/bash/manual/html_node/Bash-History-Builtins.html](https://www.gnu.org/software/bash/manual/html_node/Bash-History-Builtins.html)
- Bash documentation: [https://www.gnu.org/software/bash/manual/bash.html](https://www.gnu.org/software/bash/manual/bash.html)
- Auditd documentation: [https://linux-audit.com/](https://linux-audit.com/)
- Linux Handbook: [https://linuxhandbook.com/](https://linuxhandbook.com/)


