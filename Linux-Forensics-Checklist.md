# KIT-CERT Checklist for Linux Forensics

This document outlines [our](https://www.cert.kit.edu) initial actions to
investigate a potentially compromised Linux system. It assumes that the reader
has a good technical understanding of how Linux systems work. This is basically
a copy/paste list of things to do in a certain order.

Make sure you understand each action point before you reenact it..

## Preliminary Considerations

Forensic investigations of computer hardware is usually divided in two phases:
online forensics (analysis of the running system) and offline forensics
(examination of the permanent storage).

This document's primary focus is the data gathering aspect of the first phase.
We assume that the reader has root access to the compromised machine.

## Find a proper place to store your findings

Every action that interacts with the storage subsystem can potentially destroy
evidence (both data and metadata). Mounting external storage changes the
contents of `/etc/mtab` and the timestamps of the containing directory `/etc`.
Merely looking at the file (`cat /etc/mtab`) changes the access time of `/etc`.

### Pushing data onto the network

You may push your findings directly onto the network, thus preventing/minimizing
changes to the local filesystems. This only works if the compromized machine is
still able to make outgoing connections to the destination server .

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

Use [cryptcat](http://cryptcat.sourceforge.net) if it's available on the
compromised machine.

To copy files, use `cat`:

```
cat /usr/bin/rootkit_0.1 | nc …
```

Use `dd` to  transfer whole blockdevices:
```
dd if=/dev/sdx23 | nc…
```

Using [fuse sshfs](http://fuse.sourceforge.net/sshfs.html) is discouraged for
two reasons. First, it touches lots of files ($HOME/.ssh/*, /etc). And more
importantly: attackers often change the ssh binaries to intercept passwords.

The same problems apply to the `… | ssh user@host 'cat > /my/destination/file`
approach.

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
they touch lots of files. Check for empty pre-existing `tmpfs`-filesystems.

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
ps wwwe -A -o pid,ppid,sess,tname,tpgid,comm,f,uid,euid,rgid,ruid,gid,egid,fgid,\
ouid,pgid,sgid,suid,supgid,suser,pidns,unit,label,time,lstart,lsession,seat,\
machine,ni,wchan,etime,%cpu,%mem,cgroup:65535,args:65535 > ps_dump_e.txt
ps www -A -o pid,ppid,sess,tname,tpgid,comm,f,uid,euid,rgid,ruid,gid,egid,fgid,\
ouid,pgid,sgid,suid,supgid,suser,pidns,unit,label,time,lstart,lsession,seat,\
machine,ni,wchan,etime,%cpu,%mem,cgroup:65535,args:65535 > ps_dump.txt
```

Something more human-readable:
```sh
pstree -a -l -p -u    > pstree_alpu.txt
pstree -a -l -p -u -Z > pstree_alpuZ.txt
```

`lsof` has the most unstable commandline interface. We're planning to include versions for specific Linux distributions in the future…

```sh
lsof -b -l -P -X -n -o -R -U > lsof_blPXnoRU.txt
```

```sh
# time pid creator limits
for i in t p c l; do ipcs -a -${i} > ipcs_a_${i}.txt;done
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
for s in $(loginctl list-sessions --no-legend | awk '{print $1}'); do
    loginctl show-session ${s} > loginctl_show-session_${s}.txt;
done
for u in $(loginctl list-users --no-legend | awk '{print $1}'); do
    loginctl show-user ${u} > loginctl_show-user_${u}.txt;
done
```

### System State and Configuration

```sh
dmesg > dmesg.txt
cat /proc/mounts > proc_mounts.txt
# or use the all-namespace-encompassing version
for p in $(md5sum /proc/mounts /proc/*/mounts | sort | uniq -d -w 32 | awk '{print $2}'); do
    cat $p > ${p////_};
done
cat /proc/mdstat > proc_mdstat.txt
lspci > lspci.txt
uname -a > uname_a.txt
uptime > uptime.txt
```

### Dumping suspicious processess

Have a closer look at the process list. Do this for every suspicious process
(assign pid to the `PID` variable beforehand):
```sh
# insert correct PID here!
export PID=12345
```

Stop the process:
```sh
kill -STOP ${PID}
```

Note: While the reception of a  `SIGSTOP` cannot be prevented by the receiving
process, it can be observed from the outside. This might cause a controlling
parent-process to react accordingly. Modern systems provide an alternative method via
[cgroups](https://www.kernel.org/doc/Documentation/cgroups/cgroups.txt),
especially the [freezer
subsystem](https://www.kernel.org/doc/Documentation/cgroups/freezer-subsystem.txt).
There is one caveat: `ptrace()`ing a frozen process will block, rendering
this approach unfit for our purposes. It might nevertheless prove beneficial to
freeze everything first and then sort out the mess afterwards. We have recently
seen malware that forks children at a high frequency meking it hard to stop all
processes. This might be a way to tackle this problem; watch this space for
updates :-)

Preserve original location of executable (plus a broken symlink of file was deleted) and the contents:
```sh
ls -l /proc/${PID}/ > proc_${PID}_ls_l.txt
cat /proc/${PID}/exe > proc_${PID}_exe
```

Create a coredump to preserve the process memory:
```sh
gdb -nh -batch -ex gcore -p ${PID}
```

We have not found a way to dump the cores directly into an unnamed pipe and out
into the net. Using FIFOs does not work because gdb needs to seek within the
file while writing it. Using a tmpfs might fails because some coredumps can get
pretty big. There are no widely available and stable compressing filesystems
available for Linux at the time of writing.

Check for shared memory segments:
```sh
# look for /dev/shms
less /proc/${PID}/map
```

Save some more state information about the process. The available data in the
`/proc/$PID/` of the procfs varies between different kernel versions. Please
check `/proc/self` to what's available and adjust the next commandline
accordingly.

```sh
# use "tar -cf - /proc/… | …" to pipe the tarball to stdout
tar cf proc_${PID}.tar /proc/${PID}/{auxv,cgroup,cmdline,comm,environ,limits,\
loginuid,maps,mountinfo,sched,schedstat,sessionid,smaps,stack,stat,statm,status,\
syscall,wchan}
```

Have a look at the open files. If the file has been deleted, ls will append
` (deleted)` to the destination filename. The contents can still be accessed
using the symlinks in `/proc/${PID}/fd`. This often happens with malware
written in interpreted languages like perl and python. Save all interesting
open files now:
```sh
ls -l /proc/${PID}/fd > proc_${PID}_fd.txt
# copy interesting open files, substitute MYFD with file descriptor number
MYFD=1234
cat /proc/${PID}/${MYFD}> proc_${PID}_fd_${MYFD}
```

The online forensics parts of this guide end with us pulling power from the
machine. There's usually no need to kill the suspicious processes.

Only kill the process if
1. its presence is preventing you from continuing your work (e.g. mountpoints are busy) *and*
1. you are sure that you don't need to access it anymore *or*
1. if you have to keep the machine running after your investigation is finished.



```sh
# are you sure you want to do this?
kill -9 ${PID}
```

### Create a filesystem timeline

If you can't get an image of the system's storage afterwards for offline
forensics, you need to create a rudimentary timeline now. Otherwise, skim over
the next parts and process to [shutdown part](#power-down-system).

Using `find` on
a filesystem will `stat()` every file and directory on a filesystem thus
changing all access timestamps. There are (at least) two ways to circumvent
this.

#### Remount all filesystems to readonly

For every filesystem that we want to look at, remount it ro:
```sh
# set mountpoint
MOUNTPOINT=/home
mount -o remount,ro ${MOUNTPOINT}
```

Then create a timeline:
```sh
find "${MOUNTPOINT}" -xdev -print0 | xargs -0 stat -c "%Y %X %Z %A %U %G %n" >> timestamps.dat
```

#### Create readonly aliases

For every filesystem that we want to look at, do this:
```sh
# substitute all mountpoints
ORG_FS=/home
RO_FS=/mnt/ro_fs
mkdir ${RO_FS}
mount --bind ${ORG_FS} ${RO_FS}
mount -o remount,ro ${RO_FS}
find ${RO_FS} -xdev -print0 | xargs -0 stat -c "%Y %X %Z %A %U %G %n" >> timestamps.dat
umount ${RO_FS}
```

#### Cheap alternative: noatime

```sh
mount -o remount,noatime …
```

Use this python program (written by Leif Nixon <TODO: email>) to create a human
readable timeline:
```sh
#!/usr/bin/python
# timeline-decorator.py

import sys, time

def print_line(flags, t, mode, user, group, name):
    print t, time.ctime(float(t)), flags, mode, user, group, name

for line in sys.stdin:
    line = line[:-1]
    (m, a, c, mode, user, group, name) = line.split(" ", 6)
    if m == a:
        if m == c:
            print_line("mac", m, mode, user, group, name)
        else:
            print_line("ma-", m, mode, user, group, name)
            print_line("--c", c, mode, user, group, name)
    else:
        if m == c:
            print_line("m-c", m, mode, user, group, name)
            print_line("-a-", a, mode, user, group, name)
        else:
            print_line("m--", m, mode, user, group, name)
            print_line("-a-", a, mode, user, group, name)
            print_line("--c", c, mode, user, group, name)

```

If you cannot get the system's disks (or images thereof) afterwards, feel free
to look around some more:

* [chkrootkit]( http://www.chkrootkit.org)
* [OSSEC rootcheck](http://www.ossec.net/main/rootcheck)
* [rkhunter](http://rkhunter.sourceforge.net)
* [Trojanscan](http://www.trojanscan.org)
* [CVE-Checker](http://cvechecker.sourceforge.net/)

If applicable, compare checksums of package management with actual files:

* `debsums` (Debian-based distributions)
* `rpm -Va` (Redhat-based distributions)

### Gather potentially interesting data before shutdown

Copy (or `[dd|tar -cf -] … | …`) things that might be of interest later. There
are no hard rules here, just some general ideas:

* `/etc`
* `/root`
* `/tmp`, `/var/tmp` (generally all public writable directories)
* `/usr/local`
* `/var/log`
* `/home/*/.ssh/authorized_keys`
* sshd binary on disk and in memory (`/proc/$(pidof -s sshd)/exe`)
* anything the tools in the previous step found

TODO: journald, …

## Power down system

*Don't* do a `shutdown` or `poweroff`! Cut the power (hold power button for
several seconds) or »force off« virtual machines.

## Offline Forensics

TODO:
* Binaries: strings, hexdump, objdump, elf*, gdb, rec (http://www.backerstreet.com/rec/rec.htm), IDAPro,…
* Logfiles: grep, sort, log2timeline, …
* Autosy, rkhunter, …

#### Authors:
 * Heiko Reese <heiko.reese@kit.edu>
