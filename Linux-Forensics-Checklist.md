# KIT-CERT's Checklist for Linux Forensics

## Preliminary Considerations

Forensic investigations of computer hardware is usually divided in two phases:
online forensics (analysis of the running system) and offline forensics
(examination of the permanent storage).

This document's primary focus s the first phase (online forensics).

## Find a proper place to store your finding

Every action that interacts with the storage subsystem can potentially destroy
evidence (both data and metadata). Mounting external storage changes the
contents of `/etc/mtab` and the timestamps of the containing directory `/etc`.
