---
title: "Restrict a User to SSH Forced Command"
tags: ssh
---
We are all using `SSH` in our daily tasks. It is mostly used for the connection to the remote host, but as we know,
we are able to execute the command over the `SSH` and get the output back to our local terminal.

Sometimes we need to perform some automated routine tasks on the remote server.
Backup execution is one of the most used tasks. I'm sure you can think of an additional few tasks that could be done.
So, what is wrong with that, you may ask? The main issue for such a scenario is that once we are set an `SSH` user for such task,
it will get the full access to our remote system.

<span style="color: red">**NOT GOOD!**</span>

Fortunately, we can use a great `SSH` feature called `forced command` within the `authorized_keys` file.

The command is bound to an `SSH` key, so when the user is trying to execute a random command, the only output that will be received
is the output of the command configured previously by the admin.

### Command structure
The structure of the `forced command` within the authorized_keys file.
```
<command> <ssh public key> <comment>
```
Example:
```
command="date" ssh-rsa AAAAB3Nz..(omit output)..cMOAIubywZCB vagrant@client1
```

### Command in action
Let's try to execute a number of commands with the command "date" on the server (see example above) and see the output.
```bash
$ ssh vagrant@server1 "date"
Mon Apr 22 09:35:15 UTC 2019
```
```bash
$ ssh vagrant@server1 "ls /var/lib"
Mon Apr 22 09:35:37 UTC 2019
```
```bash
$ ssh vagrant@server1
Mon Apr 22 09:35:08 UTC 2019
Connection to server1 closed.
```
As we can see, any command including the connection attempt results in the output of the command "date", which we set as the `forced command`.

### Environment variable
The command that is passed to the remote host, stored in the environment variable - `SSH_ORIGINAL_COMMAND`.  
We could debug the behavior of the `forced command` by "echo" the command we pass to the remote host, back to us.

```
command="echo $SSH_ORIGINAL_COMMAND" ssh-rsa AAAAB3Nz..(omit output)..cMOAIubywZCB vagrant@client1
```
```
$ ssh vagrant@server1 "df -h"
df -h
```

### Multiple commands execution
The `forced command` supports only one single command per a single `SSH` key pair.  
In order to bypass this, we could place a simple bash script as the command.  
\* The example below is taken from the [SSH, The Secure Shell: The Definitive Guide book by O'Reilly].
```bash
#!/bin/sh

/bin/echo "Welcome!
Your choices are:
1       See today's date
2       See who's logged in
3       See current processes
q       Quit"

/bin/echo "Your choice:"
read ans

while [ "$ans" != "q" ]; do
   case "$ans" in
      1)
         /bin/date
         ;;
      2)
         /usr/bin/who
         ;;
      3)
         /bin/ps
         ;;
      q)
         /bin/echo "Goodbye"
         exit 0
         ;;
      *)
         /bin/echo "Invalid choice '$ans': please try again"
         ;;
   esac

   /bin/echo "Your choice:"
   read ans
done
exit 0
```

Place the script into the appropriate location and set the executable flag for the user.  
Within the `authorized_keys` file, place the path to the script.
```
command="/usr/bin/local/tasks_script.sh" ssh-rsa AAAAB3Nz..(omit output)..cMOAIubywZCB vagrant@client1
```

The output of the execution will be the following:  
**Note the output for not existing "test" command.**
```bash
$ ssh vagrant@server1
Welcome!
Your choices are:
1       See today's date
2       See who's logged in
3       See current processes
q       Quit
Your choice:
1
Mon Apr 22 12:35:37 UTC 2019
Your choice:
2
vagrant  pts/0        2019-04-22 12:24 (10.0.2.2)
vagrant  pts/1        2019-04-22 12:35 (172.28.128.14)
Your choice:
3
  PID TTY          TIME CMD
 2028 pts/1    00:00:00 test_script.sh
 2035 pts/1    00:00:00 ps
Your choice:
test
Invalid choice 'test': please try again
Your choice:
q
Connection to server1 closed.
```

### Additional SSH arguments
When required, additional connection options may be configured for the connection in order to limit the permissions of the user.  
An example of the options that may be set:
* no-agent-forwarding - Forbids authentication agent forwarding when this key is used for authentication.
* no-port-forwarding - Forbids TCP forwarding when this key is used for authentication.
* no-pty - Prevents tty allocation (a request to allocate a pty will fail).
* no-X11-forwarding - Forbids X11 forwarding when this key is used for authentication.

>Search through the authorized keys [man pages][authorized_keys_man] for more options.

The example configuration will look like the following:
```
command="df -h",no-port-forwarding,no-X11-forwarding ssh-rsa AAAAB3Nz..(omit output)..cMOAIubywZCB vagrant@client1
```

### Rsync usage
Back to our first case scenario. Use `rsync` for the backup.

The rsync could be used in both directions:
* The user backup data from the server.
* the user backup data to the server.

<ins>Backup data from the server</ins>
```
command="/usr/bin/rsync -azv --server --sender --delete /home/vagrant/backup/ ." ssh-rsa AAAAB3Nz..(omit output)..cMOAIubywZCB vagrant@client1
```
>Flags:
* a - Archive mode. Include the following flags:
  * r - Recurse into directories
  * l - Copy symlinks as symlinks
  * p - Preserve permissions
  * t - Preserve modification times
  * g - Preserve group
  * o - Preserve owner (super-user only)
  * D - Preserve device files (super-user only) and special files
* z - Compress file data during the transfer
* v - Verbose output
* server - The option is used only on the server side because of the login restriction to the rsync command
* sender - The option is used only on the server side because of the login restriction to the rsync command
* delete - Delete extraneous files from dest dirs

From the output below we can see that the files received successfully from the server.
```bash
$ rsync -azv --delete -e "ssh" vagrant@server1:/home/vagrant/backup/ backup/
receiving file list ... done
created directory backup
./
file1
file2
file3
file6
file8
total: matches=0  hash_hits=0  false_alarms=0 data=20

sent 122 bytes  received 432 bytes  1,108.00 bytes/sec
total size is 20  speedup is 0.04
```

<ins>Backup data to the server</ins>
```
command="/usr/bin/rsync -azv --server --delete . /home/vagrant/backup/" ssh-rsa AAAAB3Nz..(omit output)..cMOAIubywZCB vagrant@client1
```
>Flags:
* a - Archive mode. Include the following flags:
  * r - Recurse into directories
  * l - Copy symlinks as symlinks
  * p - Preserve permissions
  * t - Preserve modification times
  * g - Preserve group
  * o - Preserve owner (super-user only)
  * D - Preserve device files (super-user only) and special files
* z - Compress file data during the transfer
* v - Verbose output
* server - The option is used only on the server side because of the login restriction to the rsync command
* delete - Delete extraneous files from dest dirs

From the output below we can see that the files successfully copied to the server.
```bash
$ rsync -azv --delete -e "ssh" backup/ vagrant@server1:/home/vagrant/backup/
building file list ... done
created directory /home/vagrant/backup
delta-transmission enabled
./
file1
file2
file3
file6
file8

sent 378 bytes  received 188 bytes  1,132.00 bytes/sec
total size is 20  speedup is 0.04
```

For more rsync options, refer to the [man page][rsync_man].

### Conclusion
By using a simple `forces command` provided by the `SSH`, we can make our system a bit more secure.


[//]: # Reference links
[SSH, The Secure Shell: The Definitive Guide book by O'Reilly]: http://shop.oreilly.com/product/9780596000110.do
[authorized_keys_man]: http://man.he.net/man5/authorized_keys
[rsync_man]: https://linux.die.net/man/1/rsync
