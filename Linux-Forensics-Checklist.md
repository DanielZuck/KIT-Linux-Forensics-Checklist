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
```bash
nc -l 6789 >> logfilename.txt
```

To send the standard output of a command, simply add this
```bash
 | nc -w 2 name_or_ip_of_server 6789
```

Encrypt all data in transition to prevent eavesdropping. Simply insert
[`openssl`](https://openssl.org/) into the toolchain:
```bash
nc -l 6789 | openssl enc -aes128 -d -k supersecretpw >> log.txt
```
```bash
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

