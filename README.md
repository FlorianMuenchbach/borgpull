pull-based borg setup
=====================

# The Issue

Machines that are not able to (NAT?) or shall not be able to (permissions?)
reach a backup server but still have to be backed up using borg.
Borg allows to push data to a remote backup server, but the backup can not be
initiated by the backup server itself.

# The Solution

Instead of pushing the data from the client (machine to be backed up) to the
backup server, the data is "pulled".
This is done by initiating a reverse-SSH UDS connection from backup server
to the client and triggering the client to start a borg backup (push mode!).
Therefore, it's "pull", not pull.

The solution is based on [this part of the borg
documentation](https://github.com/borgbackup/borg/blob/master/docs/deployment/pull-backup.rst#socat)
and [this original issue](https://github.com/borgbackup/borg/issues/900).


# Setup

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


## Steps on the Backup Client

* Copy/Clone/"Install" `borgpull` into `/opt/`, make sure no one but root may
  write.
* Adapt the configuration files in `configs/client`. They may be copied to a
  different location, too.
* Install `sudo`, `borg` and `socat`, if not yet installed.
* Create user:
  ```
  # useradd -r -m -d /var/lib/borg_remote_pull -s /bin/bash borgpull
  ```
  Shell is required for login (restricted later), home only to prevent SSH from
  complaining and failing at the attempt to login and `chdir to home directory`.
* Create SSH directory for user:
  ```
  % mkdir /var/lib/borg_remote_pull/.ssh
  % chmod 0700 /var/lib/borg_remote_pull/.ssh
  ```
* Add backup server user's SSH key to
  `/etc/ssh/authorized_keys/borg_backup/backup_server_keys`.
  Not having the key in the user home prevents the user account from being able
  to access it.
  * If this directory does not exist, create it:
    `mkdir -p /etc/ssh/authorized_keys/borg_backup/backup_server_keys`
  * Set correct permissions. The accessing users must have *read* permissions for their keys.
    ```
    % chmod 0655 /etc/ssh/authorized_keys
    % chown -R root:borgpull /etc/ssh/authorized_keys/borg_backup
    % chmod -R g+Xr,o-rwx /etc/ssh/authorized_keys/borg_backup
    ```
* Restrict access via SSH for that user:
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
  ExecStart=/opt/borgpull/scripts/borgpull-client SETUP

  [Install]
  WantedBy=multi-user.target
  ```
  Enable with
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
  ExecStart=/opt/borgpull/scripts/borgpull-client BACKUP
  ```

* Reload systemd daemon: `systemctl daemon-reload`
* Add the following lines to the `sudoers` file by executing `visudo`:
  ```
  # Borg backup command, but without any parameters!
  Cmnd_Alias     BORG_BACKUP = /usr/bin/systemctl start run-borg-backup.service
  
  borgpull ALL   = (root) NOPASSWD: BORG_BACKUP
  ```
  The `""` at the end of the command prevent the user from being able to add
  further statements.
* Create the backup config files in `/etc/backup` and set the permissions:
  ```
  % chmod -R u+rwX,g-rwx,o-rwx /etc/backup
  ```


# Initialize the borg repo on the remote backup server side

## On the *backup server*

```
./borgpull SERVE
```
This starts socat with `borg serve` and runs the ssh command *with* `-N`, so it
does not acquire a tty and does not run the `ForceCommand` script:

Keep running until done!

## On the *backup client*

As root (best in a dedicated "throw-away shell"):
```
% source /etc/backup/borg_env
% borg init --encryption=repokey-blake2
% rm /run/borg/netcup.socket
```

# Read from the backup

## On the *backup server*

Start socat with `borg serve`:
```
./borgpull SERVE
```




## On the *backup client*

As root (best in a dedicated "throw-away shell"):
```
% source /etc/backup/borg_env
# Example, now all borg commands work!
% borg list
```


Once done, remember to remove the socket file.
```
% rm /run/borg/netcup.socket
```

# Start a Backup

```
% ./scripts/borgpull-server --config borgpull.conf BACKUP
```
