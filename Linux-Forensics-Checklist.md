# KIT-CERT's Checklist for Linux Forensics

## Preliminary Considerations

Forensic investigations of computer hardware is usually divided in two phases:
online forensics (analysis of the running system) and offline forensics
(examination of the permanent storage).

This document's primary focus is the first phase (online forensics).

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


