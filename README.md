# ibaraki - Integrated BAckup and Remote Archiving KIt

## Purpose

Being called by `cron` periodically (e.g. once a week), will log in to specific hosts and back up some directories from them (by using uploadable remote script).

## Requirements

* `bash` and other POSIX tools on the host Ibaraki is installed and every host being backed up,
* and any of supported archivers on hosts being backed up.

## Configuration

Configuration consists of `/etc/ibaraki/ibaraki.conf`, `/etc/ibaraki/archiver_*.conf`, and `/etc/ibaraki/host_*.conf`.

All config files are written as typical shell scripts, so `#` are comments, no spaces between variable name, `=`, and value, values with spaces should be quoted, `$` substitutions are allowed, etc.

Host configs should be created for every host to back up, and should contain its hostname after `host_`.

E.g., to back up `somehost`, you should create `host_somehost.conf`.

### ibaraki.conf

About timed, automatic, and manual modes, see below.

This is the main config file and it should consist of the following variables (all of them are mandatory, except `LOCAL_TMPDIR` if `CHECK_BOTH_WAYS` is `off`):

* `HOSTS`: Space-separated list of hosts to back up. Each host should have its corresponding config file. In timed or automatic mode hosts will be backed up in order they are specified here.
* `LOGFILE`: Log file which would be used in automatic or timed mode (when script is ran without parameters). Manual mode (when list of hosts is in command line) will use `stdout` for this.
* `SHORTLOGFILE`: Log file which would be used for logging start/finish time in automatic or timed mode. Manual mode won't use this.
* `BACKUPDIR`: Where to place backup files. Backups will go to corresponding subdirectories for each host.
* `REMOTENAME`: Ibaraki will upload its remote counterpart to remote host under `$TMPDIR` with this name.
* `LOCAL_TMPDIR`: Local (on the station where `ibaraki` is deployed) temporary directory to work in. Will be created if not exist, and then deleted if empty or `REMOVE_EVEN_IF_NON_EMPTY` is `on` (have an effect only if `CHECK_BOTH_WAYS` is `on`).
* `ATTEMPTS`: How much times to try to download compressed file from host. After exceeding this amount of attempts we will give up.
* `KEEPFILES`: How much of previous backup files to keep in timed or automatic mode. Oldest files will be deleted if there's more of them than this number. No files are deleted in manual mode.

Also it can include the following flags (`on` or `off`, default is `off` if flag is not specified or set to incorrect value):

* `ENABLE_NICE`: try to lower CPU priority for archiver command with `nice` (otherwise it might cause too much CPU load that can render host, which is being backed up at the moment, unusable).
* `ENABLE_IONICE`: try to lower IO priority for `tar` command with `ionice` (otherwise it might cause too much IO load that can render host, which is being backed up at the moment, unusable).
* `DELETE_IF_GIVEN_UP`: should we delete compressed file on remote host if we've given up downloading it.
* `CHECK_ON_HOST`: check integrity of archive on host prior to uploading it to the backup server (similar checks on backup server are performed always). Useful if host cache of RAM may be unreliable.
* `CHECK_BOTH_WAYS`: if `off`, check integrity of archive after it was uploaded to the backup server. If `on`, also copy it to `LOCAL_TMPDIR` and check it there. Useful if backup server may be unreliable.
* `REMOVE_EVEN_IF_NON_EMPTY`: remove temporary backup directories (`TMPDIR` and `LOCAL_TMPDIR`) even if it already contain files.
* `IGNORE_FAIL_ON_NON_EMPTY`: silently ignore if temporary backup directories (`TMPDIR` and `LOCAL_TMPDIR`) already contain files and thus was not deleted (have an effect only if `REMOVE_EVEN_IF_NON_EMPTY` is `off`).

Example:

```
HOSTS="host1 host2 host3"
LOGFILE="/net/backupserver/backup.log"
SHORTLOGFILE="/net/backupserver/shortlog.log"
BACKUPDIR="/net/backupserver/backups"
REMOTENAME="ibaraki_tmp.sh"
LOCAL_TMPDIR=/tmp/backup
ATTEMPTS=10
KEEPFILES=3
ENABLE_NICE=on
ENABLE_IONICE=on
CHECK_ON_HOST=off
CHECK_BOTH_WAYS=on
DELETE_IF_GIVEN_UP=off
REMOVE_EVEN_IF_NON_EMPTY=off
IGNORE_FAIL_ON_NON_EMPTY=off
```

### archiver_*.conf

Currently supported archivers: `7z`, `7za`, `old7z` (you should try it, if your `7z` does not support `-mm`), and plzip.

So there is a config file for each of them.

You may add your own. Each archiver should support packing and testing operations, and should be able to pack any file to specified archive.

Archiver definition file should consist of the following variables (all of them are mandatory):

* `ARCHBIN`: Binary of archiver to call.
* `ARCHARGS`: Archiver options to specify before all others (but after the command).
* `ARCHPACK`: Command for packing.
* `ARCHTEST`: Command for testing.
* `ARCHEXT`: Extension of archive file.

Also it can include the following flag (`on` or `off`, default is `off` if flag is not specified or set to incorrect value):

* `SPECIFY_ARCHIVE_NAME`: Should we call archiver like `... sourcefile.arc sourcefile` (if `on`; typical for `7z`, `rar`, ...), or just `... sourcefile` to get `sourcefile.arc` (if `off`; typical for `gzip`, `plzip`, ...).

Actually, archiver will be called in form:

* `$ARCHBIN $ARCHPACK $ARCHARGS [<file>.$ARCHEXT] <file>` - for packing, and
* `$ARCHBIN $ARCHTEST $ARCHARGS <file>.$ARCHEXT` - for testing.

### host_*.conf

Should contain variables specific for host (all of them are mandatory). Each host should have working `sshd` with SSH key authentication enabled.

* `HOSTNAME`: SSH hostname to connect.
* `PORT`: SSH port to connect.
* `USER`: User:group under which to log in. Remote counterpart of Ibaraki will be ran under these credentials.
* `TMPDIR`: Remote temporary directory to work in. Will be created if not exist, and then deleted if empty or `REMOVE_EVEN_IF_NON_EMPTY` is `on`.
* `DIRECTORIES`: Remote directories to backup, separated by spaces.
* `ARCHIVER`: Which archiver to use to pack data. Corresponding `archiver_*.conf` should be in `/etc/ibaraki`, and archiver binary should be available on remote host.
* `SSHKEY`: Private key file to authenticate on remote host.

Example:

```
HOSTNAME=somehost.local.lan
USER=user:group
PORT=22 # Default for ssh
TMPDIR=/tmp/backup
DIRECTORIES="/etc /export /root"
ARCHIVER=7za
SSHKEY=/etc/ssh/keys/$HOST.pk
```

## Install

Generic procedure is:

```
cp ibaraki /usr/bin
ln -s /usr/bin/ibaraki /etc/cron.weekly/ibaraki
mkdir -p /etc/ibaraki
cp *.conf /etc/ibaraki
```

You may alter it for your own needs.

## Run

Generally, you're supposed to just add `ibaraki` to crontab, or make a symlink of it in `/etc/cron.weekly` or wherever (so-called timed mode).

You also can run `ibaraki` to backup all hosts immediately (so-called automatic mode), or `ibaraki <hostname>...` to backup manually only specified hosts of those in `ibaraki.conf` (so-called manual mode).

## Environment variables

Ibaraki can recognize the following variables:

* `DEBUG`: default `off`. If `on`, produce some additional debug.
* `CONFIGDIR`: default `/etc/ibaraki`. Where to search `ibaraki.conf` and other config files.

## Trivia

Named after [Kasen Ibaraki](https://en.touhouwiki.net/wiki/Kasen_Ibaraki) from Touhou Project, who is mighty Oni and supervises Hakurei Shrine and sometimes the whole Gensokyo, making it safe. Maybe `ibaraki` will make safe all your hosts by backing them up.
