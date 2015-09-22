# KIT-CERT's Checklist for Linux Forensics

## Preliminary Considerations

Forensic investigations of computer hardware is usually divided in two phases:
online forensics (analysis of the running system) and offline forensics
(examination of the permanent storage).

This document's primary focus is the first phase (online forensics). We assume
that the reader has root access to the compromised machine.

## Find a proper place to store your findings

Every action that interacts with the storage subsystem can potentially destroy
evidence (both data and metadata). Mounting external storage changes the
contents of `/etc/mtab` and the timestamps of the containing directory `/etc`.
Merely looking at the file (`cat /etc/mtab`) changes the access time of `/etc`.

### Pushing data onto the network

You may push your findings directly onto the network, thus preventing/minimizing
changes to the local filesystems. This only works if the compromized machine is
still able to connect to your server.

Open a listener on your server:
```sh
nc -l 6789 >> logfilename.txt
```

To send the standard output of a command, simply add this
```sh
 | nc -w 2 name_or_ip_of_server 6789
```

Encrypt all data in transition to prevent eavesdropping. Simply insert
[`openssl`](https://openssl.org/) into the toolchain:
```sh
nc -l 6789 | openssl enc -aes128 -d -k supersecretpw >> log.txt
```
```sh
 | openssl enc -aes128 -e -k supersecretpw | nc -w 2 name_or_ip_of_server 6789
```

Use [cryptcat](http://cryptcat.sourceforge.net) if it's already available on
the target machine.

To copy files, use `cat`:

```
cat /usr/bin/rootkit_0.1 | nc …
```

Use `dd` to  transfer whole blockdevices:
```
dd if=/dev/sdx23 | nc…
```

### Collecting data on local storage

If you decide to collect your findings locally, please refrain from using
existing storage of the compromised system. There are two viable options:
external storage like USB-sticks or memory-backed filesystem aka `tmpfs`.
Please save a listing of all mounts in all namespaces before mounting anything.

Check all the different mounts:
```
md5sum /proc/mounts /proc/*/mounts | sort | uniq -d -w 32
```

Get creative to solve this chicken-egg-problem! If you have copy/paste on your
console, simply `cat`the files and copy the aferwards. Don't use screen/tmux,
the touch lots of files. Check for empty pre-existing `tmpfs`-filesystems.

Find a proper location for the mountpoint and Mount your device:
```
mount -t tmpfs none /mnt
# or
mount /dev/sdx1 /mnt
```

## Collecting evidence

Collect evidence by saving potentielly interesting parts of the system state.
Start with the most volatile and work your way down:

1. network and connection state
1. process state
1. users
1. system state and configuration

The following commands assume that you are writing your findings to a local
storage and that your current working directory is set accordingly.

Some programs have rather unstable commandline parameters, please adjust
accordingly (if possible, use `--help` instead of the manpage to find out). You
can find the long versions (if applicable) as comments above every command.

Some modern Linux systems have SELinux enabled. Run `getenforce` to find out if
SELinux is enforcing, permissive, or disabled. If the state is enforcing, we
need to get selinux information when applicable. Most tools provide a switch
`-Z` for that. Such commands are marked with a special comment like
`# SELinux: add "-Z"`.

### Network state

Get state of existing connections and open sockets:
```sh
# --verbose --wide --extend --timers --program --numeric (--listening)
netstat -v -W -e -o -p -n     > netstat_vWeopn.txt
netstat -v -W -e -o -p -n -l  > netstat_vWeopnl.txt
# same without --numeric
netstat -v -W -e -o -p        > netstat_vWeop.txt
netstat -v -W -e -o -p -l     > netstat_vWeop.txt
```

Redo using `ss` if available:

```sh
# --options --extended --processes --info --numeric (--listening )
ss -o -e -p -i -n    > ss_oepin.txt
ss -o -e -p -i -n -l > ss_oepinl.txt
# same without --numeric
ss -o -e -p -i       > ss_oepi.txt
ss -o -e -p -i -l    > ss_oepil.txt
```

Dump arp cache:
```sh
arp -n > arp_n.txt
ip neigh show > ip_neigh_show.txt
```

Get routing-related stuff:
```sh
for i in link addr route rule neigh ntable tunnel tuntap maddr mroute mrule; do
    ip $i list > ip_${i}_l.txt;
done
```

Capture iptable's state:
```sh
# --verbose --numeric --exact --list --table
for t in filter nat mangle raw; do iptables -v -n -x -L -t > iptables_vnxL_t${t}.txt; done
for table in filter mangle raw; do ip6tables -n -t ${table} -L -v -x > ip6tables_nt_${table}.txt; done
for table in filter nat broute; do ebtables -L --Lmac2 --Lc -t ${table} > ebtables_L_Lmac_Lc_t_${table}.txt; done
```

### Process State

Save process table:

```sh
ps auxwwwe > ps_auxwwwe.txt
```

Formatting of certain columns seems to be broken in many versions of `ps`, so
we have to add the :xxxxx-postfixes to enforce wide columns. This is not meant
for human consumption:

```sh
ps wwwe -A -o pid,ppid,sess,tname,tpgid,comm,f,uid,euid,rgid,ruid,gid,egid,fgid,ouid,pgid,sgid,suid,supgid,suser,pidns,unit,label,time,lstart,lsession,seat,machine,ni,wchan,etime,%cpu,%mem,cgroup:65535,args:65535 > ps_dump_e.txt
ps www -A -o pid,ppid,sess,tname,tpgid,comm,f,uid,euid,rgid,ruid,gid,egid,fgid,ouid,pgid,sgid,suid,supgid,suser,pidns,unit,label,time,lstart,lsession,seat,machine,ni,wchan,etime,%cpu,%mem,cgroup:65535,args:65535 > ps_dump.txt
```

Something more human-readable:
```sh
pstree -a -l -p -u    > pstree_alpu.txt
pstree -a -l -p -u -Z > pstree_alpuZ.txt
```

`lsof` has the most unstable commandline interface. We're planning to include versions for specific linux distributions in the future…

```sh
lsof -b -l -P -X -n -o -R -U > lsof_blPXnoRU.txt
```

```sh
# time pid creator limits
 in t p c l; do ipcs -a -${i} > ipcs_a_${i}.txt;done
```

Add this on systems that use [systemd](http://www.freedesktop.org/wiki/Software/systemd/):
```sh
systemctl status -l > systemctl_status_l.txt
```

### Users

```sh
last > last.txt
lastlog > lastlog.txt
who > who.txt
w > w.txt
```

Add this on systems that use systemd:
```sh
loginctl list-sessions > loginctl_list-sessions.txt
for s in $(loginctl list-sessions --no-legend | awk '{print $1}'); do loginctl show-session ${s} > loginctl_show-session_${s}.txt; done
for u in $(loginctl list-users --no-legend | awk '{print $1}'); do loginctl show-user ${u} > loginctl_show-user_${u}.txt; done
```

### System State and Configuration

```sh
dmesg > dmesg.txt
cat /proc/mounts > proc_mounts.txt
# or use the all-namespace-encompassing version
for p in $(md5sum /proc/mounts /proc/*/mounts | sort | uniq -d -w 32 | awk '{print $2}'); do cat $p > ${p////_}; done
cat /proc/mdstat > proc_mdstat.txt
lspci > lspci.txt
uname -a > uname_a.txt
uptime > uptime.txt
```

# Dumping suspicious processess

Have a closer look at the process list. Do this for every suspicious process:

Assign it to the `PID` variable for later usage:
```sh
# insert correct PID here!
export PID=12345
```

Stop the process:
```sh
kill -STOP ${PID}
```





#### Authors:
 * Heiko Reese <heiko.reese@kit.edu>
 * Tobias Dussa <tobias.dussa@kit.edu>
