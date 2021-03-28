# circular-snapshot

A set of scripts that provide `rsnapshot`-like functionality with a circular
buffer system for backup folders. Written in `bash`; your system must have
`basename` and `realpath` installed.

This script makes backups a lot faster for large backup sets containing lots of
files with low churn; it was developed for SCCS's Ibis server which has over 1TB
of data with millions of files, most of which haven't changed in decades. Most of
`rsnapshot`'s time was spent deleting the oldest daily backup and making a new
copy to backup into.  Instead of deleting the oldest backup, these scripts use
it as a base for syncing the newest backup on top of, in a ring-buffer sort of
format.

## Installation

Put `circular-snapshot` and `circular-snapshot-rot` somewhere in your `$PATH`.

## Usage and Config

Ther are two scripts: `circular-snapshot` and `circular-snapshot-rot`.
`circular-snapshot` syncs files and script output to the backup folder (and
should be called on every backup), while `circular-snapshot-rot` rotates files
for higher backup intervals. They probably both should be invoked with `cron` at
regular intervals:

```
# create a daily snapshot at midnight every day
00 00 * * * root /usr/local/sbin/circular-snapshot daily /backups
# create a weekly backup snapshot from the daily snapshot at 12:00 every Monday
00 12 * * 1 root /usr/local/sbin/circular-snapshot-rot daily weekly /backups
```

Note that `circular-snapshot` and `circular-snapshot-rot` will refuse to run at
the same time in the same backup folders, so offsetting your higher rotations
from your main backup is a good idea.

### `circular-snapshot`

```
circular-snapshot [-s | --sources file-list] [-f --filter filter-file]
                  [-t | --scripts script-list] [-r | --rotations num-rots]
                  <level> <backup-dir> [source1 source2 ...]
```

`circular-snapshot` syncs all files and directories specified in one of two ways:

- On the command line in the `source1`, `source2`, etc. arguments
- From a file (defaults to `/etc/circular-snapshot.txt`, but can be specified
  with the `--sources` option) containing a list of files or directories to back
  up, with each file or directory on a single line. (Blank lines are ignored, as
  well as lines beginning with `#`.)

The sources file does not support wildcards, etc.; if more complicated exclusion
patterns are desired, additional filter rules can be provided directly to
`rsync` through a file specified with the `--filter` option. See the [rsync man
page](https://man7.org/linux/man-pages/man1/rsync.1.html#FILTER_RULES) for more
information on filter rules. 

The `level` argument specifies the backup level (e.g. `daily`, `weekly`); this
can be any valid directory name and has no effect on the functionality.
Obviously if you want to have additional snapshots using `circular-snapshot-rot`
the `level` argument must match the `from-level` argument of that script so that
it knows where to pull from.

The `backup-dir` argument is pretty self-explanatory: all your backups will go
into this directory.

The `--rotations` option specifies the number of previous backups to keep
(defaulting to 3); note that this number is zero-indexed, so `--rotations=2`
with a level of `daily` will keep three backups: `daily.0`, `daily.1`, and
`daily.2`. (Each backup uses hardlinks, so identical files are effectively
deduplicated; as such the storage usage of a given number of backup rotations
isn't linearly proportional to the number of backups.)

`circular-snapshot` can also sync the output of scripts, such as database
backups, etc. Scripts are listed in a file provided with the `--scripts` option
with a similar format to the sources file. Each script is run in a temporary
directory, and any files present in that directory after script execution are 
synced to the backup folder as described below.

### `circular-snapshot-rot`

```
circular-snapshot-rot [-r | --rotations num-rots] <from-level> <to-level>
                      <backup-dir>
```

`circular-snapshot-rot` rotates your backups. It uses a similar ring-buffer
system but syncs from the latest backup of its `from-level` argument into its
`to-level` backups. For instance, `circular-snapshot-rot daily weekly /backups`
would sync the contents of `/backups/daily.0` to `/backups/weekly.0`.

The `--rotations` option works just the same as in `circular-snapshot`.

## Output

### Backup Files

The backup output of these scripts when properly configured is almost identical
to standard `rsnapshot`. If your lowest backup set is `daily`, followed by
`weekly`, your provided backup folder will contain `daily.0/`, `daily.1/`, etc.
as well as `weekly.0/`, `weekly.1`, etc. A minor difference is that `rsnapshot`
when rotating e.g. a daily to a weekly backup will perform a move operation
which moves `daily.3` (or whatever your highest backup rotation is) to
`weekly.0`; the `circular-backup-rot` script will copy changes over and not
delete `daily.3`.

The output of scripts specified with `--scripts` will each be moved into a
subfolder of the `script-backup` folder at the root of the backup (e.g.
`daily.0/script-backup/`). Each folder will be equal to the name of the scripts
file minus any extension (for example, the output of `foo/bar/data-backup.sh`
would be placed in `daily.0/script-backup/data-backup`).

Both scripts place a lockfile `circular-backup.lock` in the backup directory to
avoid clashing with each other when operating on the same backups. If they crash
mid-backup for whatever reason, the lockfile might still exist and can be
manually removed.

### Logs

Rudimentary log output is printed to stdout, redirect this to a logfile or
something, I don't care.
