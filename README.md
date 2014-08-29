#ZFS tools

| Donate to support this free software |
|:------------------------------------:|
| <img width="164" height="164" title="" alt="" src="doc/bitcoin.png" /> |
| [1Cw9nZu9ygknussPofMWCzmSMveusTbQvN](bitcoin:1Cw9nZu9ygknussPofMWCzmSMveusTbQvN) |

The ZFS backup tools will help you graft an entire ZFS pool as a filesystem
into a backup machine, without having to screw around snapshot names or
complicated shell commands or crontabs.

The utilities let you do this:

1. zfs-shell:  
   a shell that allows remote ZFS administration and nothing more

3. zsnap:  
   a command that snapshots a dataset or pool, then deletes old snapshots

4. zreplicate  
   a command that replicates an entire dataset tree using ZFS replication
   streams.  Best used in combination with zsnap as in:
   
   - zsnap on the local machine
   - zreplicate from the local machine to the destination machine

   Obsolete snapshots deleted by zsnap will be automatically purged on
   the destination machine by zreplicate, as a side effect of using
   replication streams.  To inhibit this, use the
   --no-replication-stream option.
   
   Run `zreplicate --help` for a compendium of options you may use.

5. zbackup
   a command to snapshot and replicate filesystems according to their user properties.
   The following user properties define the behaviour, where *tier* is
   arbitrary, but expected to be e.g. hourly, daily, weekly, etc.
   All properties must be in the module 'com.github.tesujimath.zbackup',
   so prefix each property listed here with 'com.github.tesujimath.zbackup:',
   following the best practice for user properties as described on the zfs man page.
   - *tier*-snapshots      - turns on snapshots, and limits how many snapshots to keep in given tier
   - *tier*-snapshot-limit - limits how many snapshots to keep in given tier (overrides *tier*-snapshots)
   - replica               - comma-separated list of dstdatasetname, as used by zreplicate
   - replicate             - *tier*, which tier to replicate

   Snapshotting for a given tier will be active as soon as *tier*-snapshots is defined
   with an integer value, with a property source of local.  Received properties will not
   cause new snapshots to be taken.

   However, old snapshots will be reaped if the property source is local or
   received.  This means that reaping old snapshots on a destination replica is
   driven by the received property *tier*-snapshots, or the property
   *tier*-snapshot-limit, with the latter overriding the former if both are
   present.  Note that the limit property functions even if its source is
   inherited.

   Replication is done for a single tier only, as per the 'replicate' property.
   Again, these properties must have the source being local to have any effect.
   Note that the --no-replication-stream option for zreplicate is used, so that
   no destination replica snapshots and filesystems are deleted as a side-effect
   of running a backup.  To purge obsolete snapshots from the destination, it is
   recommended to use the behaviour described in the previous paragraph.

   Run `zbackup --help` for the usage, and complete options.

   Run `zbackup --list` to see what backup properties are set.

The repository, bug tracker and Web site for this tool is at [http://github.com/Rudd-O/zfs-tools](http://github.com/Rudd-O/zfs-tools).  Comments to me through rudd-o@rudd-o.com.

##Setting up

Setup is rather complicated.  It assumes that you already have ZFS running
and vaults on both the machine you're going to back up and the machine that
will be receiving the backup.

###On the machine to back up

- Install the zfs-shell command   
  `cp zfs-shell /usr/local/sbin`  
  `chmod 755 /usr/local/sbin/zfs-shell`  
  `chown root.root /usr/local/sbin/zfs-shell`  

- Create a user with a home directory and shell `zfs-shell`  
  `useradd -rUm -b /var/lib -s /usr/local/sbin/zfs-shell zfs`

- Let `sudo` know that the new user can run the zfs command  
  `zfs ALL = NOPASSWD: /usr/local/sbin/zfs`  
  (ensure you remove the `requiretty` default on `/etc/sudoers`)
  (check `sudoers.zfs-tools` in `contrib/` for an example)

- Set up a cron job to run `zsnap` as frequently as you want to,
  snapshotting the dataset you intend to replicate.

###On the backup machine

- Set up public key authentication for SSH so the backup machine
  may log as the user `zfs` (as laid out above) in the machine to
  be backed up.

- Create a dataset to receive the backup stream.

- Set up a cron job to fetch the dataset snapshotted by zsnap
  from the remote machine into the newly created dataset.  You
  will use `zreplicate` for that (see below for examples).

- After the first replication, you may want to set the `mountpoint`
  attributes on the received datasets so they do not automount
  on the backup machine.

###Test

If all went well, you should be able to do this without issue:

(on the machine to back up)

    [root@peter]
    zsnap senderpool

(on the machine to receive)

    [root@paul]
    zfs create receiverpool/senderpool # <--- run this ONLY ONCE
    zreplicate -o zfs@paul:senderpool receiverpool/senderpool
    # this should send the entire senderpool with all snapshots
    # over from peter to paul, placing it in receiverpool/senderpool

(on the machine to back up)

    [root@peter]
    zsnap senderpool

(on the machine to receive)

    [root@paul]
    zreplicate -o zfs@paul:senderpool receiverpool/senderpool
    # this should send an incremental stream of senderpool
    # into receiverpool/senderpool

And that's it, really.
