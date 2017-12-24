# small-freebsd-jails

These are some notes on building small FreeBSD jails, created from personal
experience. It's not an authoritative guide.

There are tools for creating jails more easily.

## mkjail, src.conf.buildworld, src.conf.installworld

Build the jail from source. Notable make variables:

- DESTDIR
- SRCCONF
- MAKEOBJDIRPREFIX

DESTDIR is DESTDIR.

SRCCONF allows for exclusion of things from base that we do not want in our
jail.

Setting MAKEOBJDIRPREFIX to something other than the default is beneficial if
we want to build once and (re-)install multiple times without having to rerun
buildworld, when /usr/obj may have been used for other, non-jail builds.

With the configuration in this git, on amd64, FreeBSD 10.3 (I should update):

````
$ du -hd0 /usr/local/jails/jail3
 62M    /usr/local/jails/jail3
````

## nullfs, tmpfs

from the manual:

````
DESCRIPTION
     The nullfs driver will permit the FreeBSD kernel to mount a loopback file
     system sub-tree.

[...]

     The tmpfs driver will permit the FreeBSD kernel to access tmpfs file sys‐
     tems.

````

We can loopback mount our jail directory as read-only, and mount /tmp and /var
as tmpfs, or memory disks (man 4 md). /var will be populated at jail startup
(see /etc/rc.d/var and populate_var).

create the ro mount-point on host:

````
mkdir -p <path-to-ro-mount>
````

/etc/fstab on host:

````
/usr/local/jails/jail3 /usr/local/jails/jail3_ro nullfs ro 0 0
tmpfs /usr/local/jails/jail3_ro/tmp tmpfs rw,size=10M 0 0
tmpfs /usr/local/jails/jail3_ro/var tmpfs rw,size=10M 0 0
````

/etc will be mounted read-only. Setting update_motd to NO in /etc/rc.conf of
the jail removes a warning at jail startup.

## /etc/jail.conf

I don't have vnet (I should update/rebuild). Networking doesn't work very
well currently.

on host:

````
jail3 {
   persist;
   path = /usr/local/jails/jail3_ro;
   mount.devfs;
   host.hostname = jail3;
   interface = lo0;
   ip4.addr = 127.0.0.4;
   exec.start = "/bin/sh /etc/rc";
   exec.stop = "/bin/sh /etc/rc.shutdown";
}
````

Explicitly set custom devfs rulesets are a good idea (man 5 devfs.rules).

## Starting the jail

````
$ jls
   JID  IP Address      Hostname                      Path
$ jail -c jail3
jail3: created
/etc/rc: WARNING: $hostname is not set -- see rc.conf(5).
Creating and/or trimming log files.
Starting syslogd.
ELF ldconfig path: /lib /usr/lib /usr/lib/compat
32-bit compatibility ldconfig path: /usr/lib32
Clearing /tmp (X related).
Starting cron.

Sun Dec 24 06:18:42 UTC 2017
$ jls
   JID  IP Address      Hostname                      Path
    28  127.0.0.4       jail3                      /usr/local/jails/jail3_ro
$ jexec jail3 /bin/sh
$ touch foo
touch: foo: Read-only file system
````

Shutting down the jail from host:

````
$ jail -r jail3
Stopping cron.
Waiting for PIDS: 26337.
.
Terminated
jail3: removed
````
