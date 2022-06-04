pull-based Borg setup
=====================

# The Issue

Machines that are not able to (NAT?) or must not be able to (permissions?
policies?) reach a backup server but still have to be backed up using
[Borg](https://www.borgbackup.org/).
Borg allows to push data to a remote backup server, but the backup can not be
initiated by the backup server itself.

# The Solution

Instead of pushing the data from the client (machine to be backed up) to the
backup server, the data is "pulled".
This is done by initiating a reverse-SSH UDS connection from backup server
to the client and triggering the client to start a Borg backup (push mode!).
Therefore, it's "pull", not pull.

The solution is based on [this part of the Borg documentation][1] and [this
original issue][2].

By adding configuration files, sanity checks and by hiding all technical details
behind a single command for running a backup, the initial idea becomes easily
reusable across multiple backup clients.


[1]: https://github.com/borgbackup/borg/blob/master/docs/deployment/pull-backup.rst#socat
[2]: https://github.com/borgbackup/borg/issues/900

## Features

* Works for backup servers behind NAT
* Does not require the client to log in to the backup server
* Server initiated backup process
* Borg configuration on backup client
* Automatic pre- and post-backup script execution

## How it works (roughly...)

1. Backup server starts Borg in "serve" mode
2. Backup server establishes a remote SSH connection to the client and binds
   a local UDS socket to one that appears on the remote client side
3. The login via SSH automatically triggers the backup procedure on the client
   side (using SSH's `ForceCommand`)
4. The client can now push it's backup data through the SSH tunnel to the
   server using `borg create`
5. Backups are being rotated an pruned using `borg prune`


# Usage

## Initiate a Backup from the Backup Server

```
% ./scripts/borgpull-server --config <CONFIG>  BACKUP
```

> :warning: **Always check your backups for completeness and consistency** :warning:
>
> Try restoring from the backup before the original data is actually lost.
>
> Don't blindly trust (backup) scripts shared by some guy on the internet…

## Initialize the Borg Repository

This needs to be done **before** the first backup!

### On the *backup server*

```
./borgpull-server -c <CONFIG> SERVE
```
This starts socat with `borg serve` and runs the ssh command *with* `-N`, so it
does not acquire a tty and does not run the `ForceCommand` script:

Keep running until done!

### On the *backup client*

As root (best in a dedicated "throw-away shell", enviornment...):
```
% source /etc/borgpull/borg_env
% borg init --encryption=repokey-blake2
% rm /run/borg/borgpull.socket
```

## Read / Restore from the backup

### On the *backup server*

Start socat with `borg serve`:
```
./borgpull-server -c <CONFIG> SERVE
```

### On the *backup client*

As root (best in a dedicated "throw-away shell", enviornment...):
```
% source /etc/borgpull/borg_env
```

Now, all Borg commands will work on the client!
For example, to list the available backups:
```
% borg list
```

## Manually trigger a backup from the client side

### On the *backup server*

Borg has to be running in "serve" mode:
```
./borgpull-server -c <CONFIG> SERVE
```

### On the *backup client*

As root:
```
% ./borgpull-client -c /etc/borgpull SETUP
% ./borgpull-client -c /etc/borgpull BACKUP
```

# Setup

**Note:** Based on ArchLinux. Should not matter too much, tho.

## Prerequisites / Required packages

Both, the backup server and the client, need the following packages to be
available:

* `borg` (well, obviously...)
* `ssh`
* `socat`

## Steps on Backup Server

* Create backup server `borg` user
  ```
  # useradd -r -m -d /var/lib/borg_remote_pull -s /bin/nologin borg
  ```
* Create SSH key for `borg`:
  ```
  # sudo -u borg ssh-keygen -t ed25519
  ```
* Copy the sample config from `configs/server/borgpull.conf.example` to
  `borgpull.conf` (any location that suits) and adapt

Note: If a host entry from the SSH config should be used, it has to be added
tot the SSH config of the `borg` user at
`/var/lib/borg_remote_pull/.ssh/config`.


# Configuration

For both, server and client, matching configuration is required.

## Server Configuration

The server side requires one configuration file per client.
A sample configuration file can be found
[here](configs/server/borgpull.conf.example).

**Important:**
The socket path (`REMOTE_SOCKET_PATH`) on the server side must match the
`BACKUP_SOCKET` in the `borg_env` file on the client side.

**Hint:**
Choose different socket paths for different clients to avoid conflicts.

## Client Configuration

The configuration on the client side is split into a set of different files.

* `borg_env`

   Basic borgpull settings. Must be configured first.
   The file contains some explanatory comments.

   This file will be `source`d.

   **Hint**: If needed, the file may contain further Borg variables or commands.
* `borg_paths`

  List of paths to be archived. Will be appended to `borg create` as `PATH...`
  argument.

  See: `borg create --help`
* `borg_patterns`

  Include pattern paths, passed to `--pattern-from` during `borg create`.

  See: `borg create --help`


* `borg_excludes`

  Exclude paths, passed to `--exclude-from` during `borg create`.

  See: `borg create --help`
* `borg_hook_pre`

   Script file to be executed before the backup is executed.
   Can be used to execute backup preparation. This can be anything from stopping
   running services and containers to running database dumps.

   An exit status code other than 0 will skip the actual backup but still execute
   the `borg_hook_post` hook.
* `borg_hook_post`

   Script file to be executed after the backup process has terminated
  (regardless of its success).
  Counterpart to `borg_hook_pre` that my be used to restart services, etc.
* `borg_options_create`

  List of additional options passed to `borg create`.

  See: `borg create --help`
* `borg_options_prune`

  List of additional options passed to `borg prune`.

  See: `borg prune --help`


## Steps on the Backup Client

* Copy/Clone/"Install" `borgpull` into `/opt/`, make sure no one but root may
  write.
* Create the backup config files (copy from `/opt/borgpull/configs/client` in
  `/etc/borgpull` and set the permissions:
  ```
  % chmod -R u+rwX,g-rwx,o-rwx /etc/borgpull
  ```
* Adapt the configuration files in `/etc/borgpull`
* Install `sudo`, `borg` and `socat`, if not yet installed.
* Create user:
  ```
  # useradd -r -m -d /var/lib/borg_remote_pull -s /bin/bash borgpull
  ```
  Shell is required for login (restricted later), home only to prevent SSH from
  complaining and failing at the attempt to login and `chdir to home directory`.
* Create SSH directory for user:
  ```
  % mkdir -m 0700 /var/lib/borg_remote_pull/.ssh
  ```
* Add backup server user's SSH key to
  `/etc/ssh/authorized_keys/borg_backup/backup_server_keys`.
  Not having the key in the user home prevents the user account from being able
  to access it.
  * If this directory does not exist, create it:
    `mkdir -p /etc/ssh/authorized_keys/borg_backup`
  * Set correct permissions. The accessing users must have *read* permissions
    for their keys.
    ```
    % chmod 0655 /etc/ssh/authorized_keys
    % vi /etc/ssh/authorized_keys/borg_backup/backup_server_keys
    % chown -R root:borgpull /etc/ssh/authorized_keys/borg_backup
    % chmod -R g+Xr,o-rwx /etc/ssh/authorized_keys/borg_backup
    ```
* Restrict access via SSH for that user (append to `/etc/ssh/sshd_config`):
  ```
  Match User borgpull
      AuthorizedKeysFile /etc/ssh/authorized_keys/borg_backup/backup_server_keys
      AllowAgentForwarding no
      AllowTcpForwarding remote
      AllowStreamLocalForwarding remote
      X11Forwarding no
      ForceCommand sudo systemctl start run-borg-backup.service
      PermitUserRC no
      StreamLocalBindMask 0177
  ```
  Note: If an `AllowUsers` policy is in place, it has to be extended, too.
* Restart sshd: `systemctl restart sshd`
* Create a systemd unit for setting up required directories, etc in
  `/etc/systemd/system/borg-backup-create-directory.service`:
  ```
  [Unit]
  Description=Creates the required directories for run-borg-backup.service

  [Service]
  Type=oneshot
  ## Stay alive for other run-borg-backup.service to acknowledge 
  RemainAfterExit=yes
  ExecStart=/opt/borgpull/scripts/borgpull-client -c <PATH_TO_CONFIGS> SETUP

  [Install]
  WantedBy=multi-user.target
  ```
  Replace `<PATH_TO_CONFIGS>` with respective directory path.
* Enable with
  ```
  % systemctl daemon-reload
  % systemctl enable --now borg-backup-create-directory.service
  ```
* Create a systemd unit for running the backup in
  `/etc/systemd/system/run-borg-backup.service`. Leads to automatic inclusion
  in journals.
  ```
  [Unit]
  Description=Starts the borg backup runner
  # Requisite does not start, if started already -- unlike Requires
  Requisite=borg-backup-create-directory.service
  After=borg-backup-create-directory.service
  
  [Service]
  Type=oneshot
  ExecStart=/opt/borgpull/scripts/borgpull-client -c <PATH_TO_CONFIGS> BACKUP
  ```
  Replace `<PATH_TO_CONFIGS>` with respective directory path.
* Reload systemd daemon: `systemctl daemon-reload`
* Add the following lines to the `sudoers` file by executing `visudo`:
  ```
  # Borg backup command, but without any parameters!
  Cmnd_Alias     BORG_BACKUP = /usr/bin/systemctl start run-borg-backup.service
  
  borgpull ALL   = (root) NOPASSWD: BORG_BACKUP
  ```

**Before the first backup can be created, the [repository must be
initialized](#initialize-the-borg-repository)**
