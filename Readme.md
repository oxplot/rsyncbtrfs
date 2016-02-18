rsyncbtrfs is a small shell script that does incremental backups under
linux using [rsync][] on [btrfs][] partitions.

Requirements
============

You need to have [rsync][] and [btrfs-tools][] installed on your system.

Setup
=====

rsyncbtrfs is a standalone shell script. You can just copy it to a
directory in your PATH:

    $ mkdir -p ~/bin
    $ curl 'https://raw.githubusercontent.com/oxplot/rsyncbtrfs/master/rsyncbtrfs' > ~/bin/rsyncbtrfs
    $ chmod a+x ~/bin/rsyncbtrfs

Example
=======

Best way to demo how to use rsyncbtrfs is with an example. Let's say we
have a btrfs formatted backup partition mounted under `/backup`. We want
to backup `/` and `/home/joe` separately to `/backup`.

First, we structure the backup partition:

    $ mkdir /backup/sys
    $ mkdir /backup/joe

Next, we need to initialize each backup directory:

    $ rsyncbtrfs init /backup/sys
    $ rsyncbtrfs init /backup/joe

All this does is to create an empty file called `.rsyncbtrfs` under the
given directory. `rsyncbtrfs backup` command aborts if it can't find
`.rsyncbtrfs` file in the destination. Just a safety check.

Time to backup:

    $ rsyncbtrfs backup /         /backup/sys --exclude='/home/**'
    $ rsyncbtrfs backup /home/joe /backup/joe

Simple enough. Just a note on `--exclude` option: any argument after the
destination is passed onto rsync. Here, we're excluding `/home` because
we don't want to duplicate the backup of `/home/joe`.

After the backup above, this is what `/backup` would look like:

    $ ls /backup/sys
    2014-07-17-13:06:17  cur
    $ ls /backup/joe
    2014-07-17-14:09:09  cur

The timestamped directories are btrfs subvolumes. `cur` is a symlink to
the latest timestamped subvolume, in this case the only one.

Running the backup again, this is what we get:

    $ ls /backup/sys
    2014-07-17-13:06:17  2014-07-17-18:31:00  cur
    $ ls /backup/joe
    2014-07-17-14:09:09  2014-07-17-18:32:11  cur

Notes
=====

rsyncbtrfs creates a new subvolume or snapshots an existing one under a
temp directory in destination, named `.inprog-XXXXX`. It only renames
the subvolume to its final timestamped location on success. Same applies
to updating of the `cur` symlink. If the backup fails for any reason,
nothing changes in the backup directory. rsyncbtrfs tries to cleanup
after itself but worse comes to worst, it might leave the inprog
directory behind, which should be cleaned up, although it's not
necessary.

You can use `--bind-mount` argument when backing up to instruct
rsyncbtrfs to bind mount the source directory under a temp path. This is
useful when you don't want to backup all the mount points under the
source:

    $ rsyncbtrfs backup --bind-mount / /backup/sys

The `--compress` argument enable the compression attribute on the new
snapshot directory. Whether the files are compressed or not depend on
the compression configuration of the Btrfs volume.

To take full advantage of btrfs COW functionality, `--inplace` flag of
rsync is used by default. This flag tells rsync to only write the
updated data in a file instead of creating a new copy and moving it into
place. Using this flag has several (possibly negative) side effects
which you should be aware of. Consult rsync's man page for further
details.

[rsync]: http://rsync.samba.org/
[btrfs]: https://btrfs.wiki.kernel.org/index.php/Main_Page
[btrfs-tools]: https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs
