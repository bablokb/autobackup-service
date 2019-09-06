Automatic Backup using rsnapshot/rsync
======================================

Overview
--------

This little project enables automatic backups triggered by a plugin of an
USB backup device.

An udev-rule starts a helper service after plugin, which in turn executes
rsnapshot with an appropriate backup-level.


Prerequisites
-------------

You should know how rsnapshot and it's backup-levels work - ideally, you already
have configured and used rsnapshot and it works fine if you start rsnapshot
manually.

If not, no problem: rsnapshot is simple to setup and to use (ask Google).

You also need a backup-device with a partition formatted with a Linux
filesystem (none of the FAT variants are suitable).  This partition needs
a label. You can add the label with the command

    sudo e2label /dev/sdc1 "Backup"


Installation
------------

To install the necessary project files, run

    git clone https://github.com/bablokb/autobackup-service.git
    cd autobackup-service
    sudo tools/install

This will also install rsnapshot and rsync in case they are not yet available
on your system.


Configuration
-------------

The autobackup-service uses the configuration-file `/etc/autobackup.conf`.
Here you have to set a number of variables:

  - `LABEL`: the label of your backup-device. To temporarily disable the
     backup, just set this to an empty value.
  - `SYSLOG`: set this to `1` for messages in the system-log
  - `WAIT_FOR_DEVICE`: the default is one second, you might need to increase
     this value if your system needs more time to detect all partitions
  - `daily`, `weekly`, `monthly`, `yearly`: the names of your backup-levels.
    Rsnapshot's example uses `alpha`, `beta`, `gamma`, but I prefer my own names.
  - `force_daily`: set this variable to `1` if you want multiple backups
    per day.
    **Note that this setting will trigger a *mount-backup-umount* cycle
    for every plugin of the device. To access the partition normally,
    just let the backup finish normally. Afterwards mount the partition manually
    instead of detaching the device.**

You can find an example for `/etc/rsnapshot.conf` in the file
`/etc/rsnapshot.conf.autobackup-service`. You should copy this file
and edit it to your needs. Idealy, you would only change the "retain"-statements
in the middle of the file and the directories you want to backup at
the end of the file.

It should be obvious that the values in `/etc/autobackup.conf` must match
your configuration in `/etc/rsnapshot.conf`.


Disabling automount
-------------------

If your Linux-system automounts removable media, you should disable
automounting. E.g. for Raspbian, you can use the default filemanager for this
task. If you run Gnome, you can run the script `tools/gnome-disable-automount`
(kindly provided by Don Cohoon, see
[this issue](https://github.com/bablokb/autobackup-service/issues/2)).


How it works
------------

The udev-rule `/etc/udev/rules.d/99-autobackup.rules` starts the system-service
defined in `/etc/systemd/system/autobackup@.service`. The service just runs
the script `/usr/local/sbin/autobackup`. This indirection is necessary since
scripts started directly by udev are not allowed to run too long.

The autobackup-script receives the currently plugged-in block-device as an
argument. It checks if this device contains a partition with a label that
matches the entry in `/etc/autobackup.conf`. If so, it mounts the partition.

Once mounted, the script checks the directories that correspond the the
backup-levels. If the directories are too old, the script just starts
rsnapshot passing the appropriate level as an argument.

