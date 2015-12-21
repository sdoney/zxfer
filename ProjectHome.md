Zxfer is a fork of Constantin Gonzalez's zfs-replicate, with many additional features (80%+ of code is new). In a nutshell, the aim of zxfer is to make backups, restores and transfers on ZFS filesystems able to be done with a single command, while having similar end-to-end assurance of data integrity as the ZFS filesystem itself.

Features include:

  * NEW mammoth [article](http://forums.freebsd.org/showthread.php?t=24113) suite on how to install, backup and restore a reliable FreeBSD 8.2 system!
  * [articles](http://code.google.com/p/zxfer/w/list) in the wiki on ZFS HDD based backup - no tapes!
  * installable from FreeBSD ports (sysutils/ports)
  * remote transfers via SSH in FreeBSD 8.2 and Solaris 11 Express, as of 0.9.4
  * recursive transfer of filesystems (via snapshots).
  * minimal dependencies. Runs from /bin/sh, and only requires rsync if that is used instead of zfs send/receive.
  * to transfer filesystem properties, and override specified properties (e.g. compression, copies, dedup etc.)
  * backup original properties to a file so as to be able to restore them later.
  * transfer via rsync. Useful when different snapshotting regimes are used on source and destination.
  * delete snapshots on destination not present on source, and transfer from the latest common snapshot - this allows using snapshot management programs like zfs-snapshot-mgmt and painless backups at the same time.
  * a comprehensive man page.
  * beep when done; useful for long transfers.
  * 0.9.3 is (somewhat) tested with FreeBSD 8.2, Solaris 11 Express. 0.9.0 tested to same extent with FreeBSD 8.0 and 8.1, OpenSolaris last version, Solaris 11 Express.

Note, the article on Freebsd.org will appear on the wiki here when it is ready for a second public draft.