# rsync-incremental-backup Adapted for OSX

Configurable bash scripts to send incremental backups of your data to a local or remote target, using [rsync](https://download.samba.org/pub/rsync/rsync.html). __Warning__: I have only adapted and tested the local backup script. 


## Description

These scripts does (as many as you want) incremental backups of the desired directory to another local or remote directory on a Mac computer. 
The first directory acts as a master (doesn't get modified), making copies of itself at the second directory (slave).
Then, you can browse the slave directory and get any file included into any previous backup.

Only new or modified data is stored (because it's incremental), so the size of backups doesn't grow too much.

If a backup process gets interrupted, don't worry. You can continue it in the next run of the script without data loss and without transferring previously transferred data.

In addition, there is a local backup script with special configuration, oriented to do backups for a GNU/Linux filesystem.
For example, it already has omitted temporal, removable and other problematic paths, and is meant to backup to a external mount point (at `/mnt`).


## Configuration

You can set some configuration variables to customize the script:

* `src`: Path to source directory. Backups will include it's content.
* `dst`: Path to target directory. Backups will be placed here.
* `remote`: *ssh_config* host name to connect to remote host (only for remote version).
* `backupDepth`: Number of backups to keep. When limit is reached, the oldest get deleted.
* `timeout`: Timeout to cancel backup process, if it's not responding.
* `pathBak0`: Directory inside `dst` where the more recent backup is stored.
* `partialFolderName`: Directory inside `dst` where partial files are stored.
* `rotationLockFileName`: Name given to rotation lock file, used for detecting previous backup failures.
* `pathBakN`: Directory inside `dst` where the rest of backups are stored.
* `nameBakN`: Name of incremental backup directories. An index will be added at the end to show how old they are.
* `logName`: Name given to log file generated at backup.
* `exclusionFileName`: Name given to the text file that contains exclusion patterns. You must create it inside directory defined by `ownFolderName`.
* `ownFolderName`: Name given to folder inside user's home to hold configuration files and logs while backup is in progress.
* `logFolderName`: Directory inside `dst` where the log files are stored.

All files and folders in backup (local and remote only) get read permissions for all users, since a non-readable backup is useless.
If you are worried about permissions, you can add a security layer on backup access level (FTP accounts protected with passwords, for example).
You can also preserve original files and folders permissions removing the `--chmod=+r` flag from script.
In system backup, the original permissions are preserved by default.


## Usage

### Setting up *ssh_config* (for remote version)

This script is meant to run without user intervention, so you need to authorize your source machine to access the remote machine.
To accomplish this, you should use *ssh keys* to identify you and set a *ssh host* to use them properly.

There are lots of tutorials dedicated to these topics, you can follow one of them.
I won't go into more detailed explanation on this, but here are some good references:

* [How To Set Up SSH Keys](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2)
* [OpenSSH Config File Examples](https://www.cyberciti.biz/faq/create-ssh-config-file-on-linux-unix/)

After that, you should use the `Host` value from your *ssh config file* as the `remote` value in the script.

### Customizing configuration values

You have to set, at least, `src` and `dst` (and `remote` in remote version) values.

If you want to exclude some files or directories from backup, add their paths (relative to backup root) to the text file referenced by `exclusionFileName`.

This is some information on filter rules that may help:

`/dir/` means exclude the root folder /dir
`/dir/*` means get the root folder /dir but not the contents
`dir/` means exclude any folder anywhere where the name contains dir/

Examples excluded: `/dir/`, `/usr/share/directory/`, `/var/spool/dir/`
`/var/spool/lpd/cf` means skip files that start with cf within any folder within `/var/spool/lpd`

Once configured with your own variable values, you can simply run the script to begin the backup process.

### Automating backups

Personally, I schedule it to run every day with a Launch Daemon. This way, I don't need to remember running it.

To use the provided Launch Daemon you have to follow these steps:

* On the command line change to the directory where you have checket out this README.md file to
* Copy the backup script to the the directory the Launch Daemon is expecting it: 
```sh
sudo cp rsync-incremental-backup-local /usr/local/bin
```
* Adjust the provided sample exclude.txt file
* Copy the provided sample exclude.txt to the directory the backup script is expecting it:
```sh
sudo cp exclude.txt /.rsync-incremental-backup
```
* Adjust the provided file com.example.rsync-backup.plist to your needs. The Launch Daemon is configured to write logs to /var/log/rsync-backup.log
* Copy the Launch Daemon config file:
```sh
sudo cp com.example.rsync-backup.plist /Library/LaunchDaemons
```
- load and start LaunchDaemon:
```sh
sudo launchctl load /Library/LaunchDaemons/com.example.rsync-backup.plist
sudo launchctl start /Library/LaunchDaemons/com.example.rsync-backup.plist
```


### Checking backup content

If you are using the default folder names, the newest data backup will be inside `<dst>/data`.
The second newest backup will be inside `<dst>/backup/backup.1`, next will be inside `<dst>/backup/backup.2` and so on.
Log files per backup operation will be stored at `<dst>/log`.


## Used *rsync* flags explanation

* `-a`: archive mode; equals -rlptgoD (no -H,-A,-X). Mandatory for backup usage.
* `-c`: skip based on checksum, not mod-time & size. More trustworthy, but slower. Omit this flag if you want faster backups, but files without changes in modified time or size won't be detected for include in backup.
* `-h`: output numbers in a human-readable format.
* `-v`: increase verbosity for logging.
* `-z`: compress file data during the transfer. Less data transmitted, but slower. Omit this flag when backup target is a local device or a machine in local network (or when you have a high bandwidth to a remote machine).
* `--progress`: show progress per file during transfer. Only for interactive usage.
* `--info=progress2`: show progress based on the whole transfer, rather than individual files. Only for interactive usage.
* `--timeout`: set I/O timeout in seconds. If no data is transferred for the specified time, backup will be aborted.
* `--delete`: delete extraneous files from dest dirs. Mandatory for master-slave backup usage.
* `--link-dest`: hardlink to files in specified directory when unchanged, to reduce storage usage by duplicated files between backups.
* `--log-file`: log what we're doing to the specified file.
* `--chmod`: affect file and/or directory permissions.
* `--exclude`: exclude files matching pattern.
* `--exclude-from`: same as `--exclude`, but getting patterns from specified file.

* Used only for remote backup:
	* `--no-W`: ensures that rsync's delta-transfer algorithm is used, so it never transfers whole files if they are present at target. Omit only when you have a high bandwidth to target, backup may be faster.
	* `--partial-dir`: put a partially transferred file into specified directory, instead of using a hidden file in the original path of transferred file. Mandatory for allow partial transfers and avoid misleads with incomplete/corrupt files.

* Used only for local backups:
	* `-W`: ignores rsync's delta-transfer algorithm, so it always transfers whole files. When you have a high bandwidth to target (local filesystem or LAN), backup may be faster.

* Used only for system backup:
	* `-A`: preserve ACLs (implies -p).
	* `-X`: preserve extended attributes.

* Used only for log sending:
	* `-r`: recurse into directories.
	* `--remove-source-files`: sender removes synchronized files (non-dir).



## References

This was inspired by:

* [Incremental Backups on Linux](http://www.admin-magazine.com/Articles/Using-rsync-for-Backups).
* [Rsync full system backup](https://wiki.archlinux.org/index.php/Rsync#Full_system_backup).
