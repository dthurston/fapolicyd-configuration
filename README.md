# Configuring fapolicyd on RHEL 8.3
RHEL makes application whitelisting easy!<br>
Application whitelisting efficiently prevents the execution of unknown and potentially malicious software.  Selinux provides mandatory access controls (MAC) for how an application should behave and is not concerned about where the application came from or whether it is known to the system.<br>
Fapolicyd by design cares solely about if this is a known application/library.  The combination of selinux and fapolicyd are complimentary to each other.<br>
1. Let's get started and install fapolicyd
```bash
# yum install fapolicyd
```
That was easy.
2. Now let's enable and start it!
```bash
# systemctl enable fapolicyd.service --now
```
3. Verify it's running
```bash
# systemctl status fapolicyd.service
```
4. The fapolicyd framework provides the following components:
* fapolicyd service
* fapolicyd command-line utilities
* fapolicyd YUM plugin
* fapolicyd rule language
The files are located under /etc/fapolicyd/
```bash
# ls /etc/fapolicyd/
fapolicyd.conf  fapolicyd.rules  fapolicyd.trust
```
fapolicyd.rules contains the rules followed<br>
fapolicyd.trust contains trusted files<br>
fapolicyd.conf is the daemon configuration file.  The average user should not have to change anything in this file... :sweat_smile:

5. Let's take a peek at the default ruleset.  To do this run the following command:
```bash
# fapolicyd-cli --list
```
Take a moment to observe how the rules are configured... They follow a "decision permission subject : object" recipe and are processed from top to bottom.  For more details on this see fapolicyd.rules manpage.

6. There are two ways to add programs to the fapolicyd database allow list.
In this scenario, we want fapolicyd to trust a non-privileged user's executable in /tmp.  This is a **terrible idea in the real world** and the scenario is being used because fapolicyd blocks users running executables out of /tmp by default on RHEL 8.3.<br>

First, the user tries to run their program:
```bash
$ cp /bin/ls /tmp
$ /tmp/ls
bash: /tmp/ls: Operation not permitted
```
Next, a user with privileges will have to add their program to the /etc/fapolicyd/fapolicyd.trust file.  This can be done from the command line:

```bash
# fapolicyd-cli --file add /tmp/ls
# fapolicyd-cli --update
Fapolicyd was notified
```
Now, switching back to the user, the command will work

```bash
$ /tmp/ls
<snip>
```
The file, "/tmp/ls", has simply been added to the fapolicyd.trust file (along with it's byte size and a sha256 sum).  By doing this you are telling fapolicyd to trust that file.

7. Alternatively, to **create a rule** for this we need to start fapolicyd in debug mode.  First I'm going to clean up that trust we just added.  Edit /etc/fapolicyd/fapolicyd.trust and remove the line with "/tmp/ls".  Update the database.

```bash
# vi /etc/fapolicyd/fapolicyd.trust
# fapolicyd-cli --update
Fapolicyd was notified
```
Now I'm going to stop the fapolicyd daemon.

```bash
# systemctl stop fapolicyd.service
```
Restart fapolicyd in debug mode like this:
```bash
# cd ~
# fapolicyd --debug 2> fapolicy.output &
[1] 24920
```

Switch to the unprivileged user again and try the command (you should get the error again).
```bash
$ /tmp/ls
-bash: /tmp/ls: Operation not permitted
```
Switch back to root and bring the fapolicyd process to the foreground with "fg" and hit CTRL + C to kill it.
```bash
# fg
fapolicyd --debug 2> fapolicy.output
^C
#
```
There should now be a file in your current directory called fapolicy.output.  We're going to grep this file for the command that was run:
```bash
# grep '/tmp/ls' fapolicy.output 
rule=14 dec=deny_audit perm=execute auid=1000 pid=25180 exe=/usr/bin/bash : path=/tmp/ls ftype=application/x-executable
rule=15 dec=allow perm=open auid=1000 pid=25180 exe=/usr/bin/bash : path=/tmp/ls ftype=application/x-executable
rule=15 dec=allow perm=open auid=1000 pid=25180 exe=/usr/bin/bash : path=/tmp/ls ftype=application/x-executable
```

As you can see, rule 14 hit and prevented it from executing.  Copy this line and we're going to edit it in the /etc/fapolicyd/fapolicyd.rules file. 
Rule 14 is the deny execution for anything untrusted, so we need to add it before that line.

The output from fapolicyd and the necessary rule are very similar, so creating the rule is not a big deal:
```
rule=14 dec=deny_audit perm=execute auid=1000 pid=25180 exe=/usr/bin/bash         : path=/tmp/ls ftype=application/x-executable
allow                  perm=execute                     exe=/usr/bin/bash trust=1 : path=/tmp/ls ftype=application/x-executable trust=0
```

The rule we're adding should look like this:
```bash
allow perm=execute exe=/usr/bin/bash trust=1 : path=/tmp/ls ftype=application/x-executable trust=0
```
The trust=1 and trust=0 is a boolean telling fapolicyd if the subject of the rule should be in the trust database or not.  1 is yes, 0 is no.  /usr/bin/bash is in the trust database via the signed rpm Red Hat delivered.
```bash
# vi /etc/fapolicyd/fapolicyd.rules
```

And you should have a line added that looks like this before the deny all rule:
```bash
allow perm=execute exe=/usr/bin/bash trust=1 : path=/tmp/ls ftype=application/x-executable trust=0

# Deny execution for anything untrusted
deny_audit perm=execute all : all
```
Start fapolicyd again and test it out

```bash
# systemctl start fapolicyd.service
```

```bash
$ /tmp/ls
<snip>
```
