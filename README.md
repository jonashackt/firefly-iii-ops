# firefly-iii-ops
Having a look into personal finance management with Firefly III


Stumbled upon [firefly-iii](https://firefly-iii.org/) while migrating away from Apple products (Macs, iPhones, etc. - see https://github.com/jonashackt/mac-to-linux).

I was looking for a multi-user, OpenSource-based, possibly self-hosted, personal and family finance management solution - which I can use together with my wife.

There are also other alternatives (see https://github.com/awesome-selfhosted/awesome-selfhosted#money-budgeting--management), but Firefly III seemed the most active developed solution to me.

> If you need some hints on how to find a suitable homeserver, have a look at this https://github.com/jonashackt/homeserver


# Install Firefly III with Docker

> Be sure to have Docker installed, e.g. [I described it here, how to do it on Manjaro](https://github.com/jonashackt/mac-to-linux?tab=readme-ov-file#docker) 

https://docs.firefly-iii.org/how-to/data-importer/installation/docker/

Download the docker-compose.yml and `.env` files as described. Also change the db passwords and importer url values.

Then run the compose install via:

```shell
docker compose -f docker-compose.yml up -d --pull=always
```

Now access firefly iii on http://localhost



## Open port on your server

See https://www.digitalocean.com/community/tutorials/opening-a-port-on-linux

If you have an `ufw` based firewall (like on Manjaro), simply run

```shell
sudo ufw allow 4000
```

Which also sustains reboots without further work.

If you choose the port `4000`, be sure to also change the `services:app:port` in the `docker-compose.yml` :

```yaml
services:
  app:
    image: fireflyiii/core:latest
    ...
    ports:
      - '4000:8080'
    ...
```

Run `docker-compose down` and `up` again.

Now access firefly iii can be found at http://localhost:4000



## Accessing your local networks Firefly III from an Firefly supporting App

There are mutliple Apps that are able to use the Firefly API https://docs.firefly-iii.org/references/firefly-iii/third-parties/apps/

Especially the Abacus App https://github.com/victorbalssa/abacus (available for Android AND iPhone) looks really promising and is easy to configure to use your Firefly III installation.

Download the App from your Appstore and configure your local network's firefly url. Then log in using OAuth client token, provided by firefly at http://localhost:4000/profile


## Make your homeserver available on the internet (for later app access)

There are multiple possibilities to make your homeserver accessible on the internet. See [this subreddit](https://www.reddit.com/r/synology/comments/otczia/is_there_an_actual_safe_way_to_access_my_nas_from/) for example.

They discuss VPN, reverse proxy, DynDNS or https://tailscale.com. I'd like to try the latter.

Which reads extremely great for me: https://tailscale.com/blog/how-tailscale-works & https://tailscale.com/blog/how-nat-traversal-works


### Install tailscale on Manjaro

https://tailscale.com/download/linux

Install the official client:

```shell
pamac install tailscale
```

Start the `tailscaled` daemon as a system service:

```shell
sudo systemctl start tailscaled
```

> To have tailscale start up with your homeserver booting, be sure to enable the systemctl service via

```shell
sudo systemctl enable tailscaled
```

Connect your server to tailscale.com

```shell
sudo tailscale up
```

Now use the link to authenticate your machine to your tailnet. That's it.


### Connect your (mobile) devices

To connect mobile devices, get the app from the app store and login via your IDP. Your mobile has automatically joined your tailnet.

By default tailscale MagicDNS is configured already. So if you add a machine having a hostname `myhomelab`, then all you need to access this machine is http://myhomelab! Pretty cool eh?!


### Share your homeserver (or your full tailscale tailnet) with the other Firefly III users

> First you should create a `fun` DNS name for your tailnet. This can be done [in the `DNS` settings as described here](https://tailscale.com/kb/1217/tailnet-name#creating-a-fun-tailnet-name). With this you have a memorable url name rather than just a random number.ts.net

If it's your wife, you might just want here to join your tailnet anyways. If it's your kid, you might restrict access of devices a bit more. 

To do the first, head over to https://login.tailscale.com/admin/users and click on `Invite external users` in the Invite users tab. Now share the link using mail or the link directly.

#### When your wife can't see your devices (only her's)

You need to reauthenticate via `Sign in with other` and not with the button `sign in with google`, but using your wife'S google adress. See https://www.reddit.com/r/Tailscale/comments/15iornp/comment/k4terws/



# Backup Firefly III

https://docs.firefly-iii.org/how-to/firefly-iii/advanced/backup/ & https://github.com/firefly-iii/firefly-iii/issues/4270

There's a backup script [which can be downloaded as GitHub gist](https://gist.github.com/dawid-czarnecki/8fa3420531f88b2b2631250854e23381). Download it to the folder where your `docker-compose.yml` and `.env` files reside and enable it's execution:

```shell
chmod +x firefly-iii-backuper.sh
```

> I got the warning `[WARNING]  The following files were not found in /home/homepike/firefly: .*.env. Skipping.
`
> To add the `.env` files all correctly to the `.tar`, I changed the third line of the script as follows:

```shell
# From
files_to_backup=(.*.env docker-compose.yml )

# to this
files_to_backup=(.env .db.env .importer.env docker-compose.yml )
```

Now all files got correctly backuped.

First try to execute the script from your firefly dir and create a example backup:

```shell
bash firefly-iii-backuper.sh backup $(date '+%F').tar
```

You can extract the `.tar` and have a look into it. It should contain the `docker-compose.yml`, versions and especially the `firefly_db.sql` file. In order to verify everything went correctly, try to restore the backup immediately after the backup ran:

```shell
bash firefly-iii-backuper.sh restore 2024-08-06.tar
```

It should print something like this:

```shell
[INFO]  Restoring the following files: .db.env .importer.env docker-compose.yml
[WARNING]  The upload volume exists. Overwriting.
[INFO]  Restoring database
```


## Automatically encrypted backups to Cloud Provider using openssl, rclone, systemd Timers & Google Drive

Backing up your database is great, but you should also place the backups on another machine to mitigate hardware failure. 

But right before uploading our backups to a Cloud Provider, we should encrypt them.


### Encrypting the `.tar` with openssl

But before syncing our backup with a Google Drive, we should encrypt it, right. Otherwise the whole homeserver thingy wouldn't make any sense. The simplest idea was to encrypt the already tared `.tar`. There are [several possibilities on how to encrypt our a `.tar` file](https://stackoverflow.com/questions/57817073/how-to-encrypt-after-creating-tar-archive), I chose `openssl` here:

```shell
openssl enc -aes-256-cbc -md sha512 -pbkdf2 -iter 1000000 -salt -pass pass:mypassword -in 2024-08-06.tar -out 2024-08-06.tar.openssl
```

This will also omit the warning `*** WARNING : deprecated key derivation used. Using -iter or -pbkdf2 would be better.` [as described here](https://unix.stackexchange.com/a/507132/140406).

Unecryption is possible adding the `-d` parameter to the command:

```shell
openssl enc -d -aes-256-cbc -md sha512 -pbkdf2 -iter 1000000 -salt -pass pass:mypassword -in 2024-08-06.tar -out 2024-08-06.tar.openssl
```

> Only be sure to have the `-salt` parameter added also, since otherwise I ran into the infamous `"error reading input file" and "bad magic number"` errors.

We also need to have an unintended execution of the `openssl` command in place, since we want to execute it from within a cron job later. Therefore we use the `pass file:myfile.txt` parameter instead like this:

```shell
# Create password file & insert your password
nano fireflybackup.txt

# encrypt
openssl enc -aes-256-cbc -md sha512 -pbkdf2 -iter 1000000 -salt -pass file:fireflybackup.txt -in 2024-08-06.tar -out 2024-08-06.tar.openssl

# decrypt
openssl enc -d -aes-256-cbc -md sha512 -pbkdf2 -iter 1000000 -salt -pass file:fireflybackup.txt -in 2024-08-06.tar.openssl -out 2024-08-06-decrypted.tar
```

This will produce an encrypted `2024-08-06.tar.openssl`, which we can now sync with cron to our Google Drive.



## Use rclone to sync encrypted tar to Google Drive

If you want to use your Google Drive, you could use [the Gnome Online Service connection described here](https://github.com/jonashackt/mac-to-linux/blob/main/README.md#google-drive-desktop-file-sync). BUT I wouldn't recommend, it wasn't a stable solution for unattendet backups at least for me.

So I looked for an alternative (I already stumbled upon it, but didn't have the time to look into it): https://github.com/rclone/rclone

First you need to install and configure rclone.

Install on Manjaro via:

```shell
pamac install rclone
```

In order to configure Google Drive with rclone, just have a look into the docs: https://rclone.org/drive/

Start with running:

```shell
rclone config
```

An interactive config wizard will guide you.

Be sure to create your own OAuth Client ID & key as stated in https://rclone.org/drive/#making-your-own-client-id (which is rather nasty to do and Google also introduced a new verification step, where they really review the application, which can take weeks :( But the good thing is: the OAuth Client ID & Key will work right away)

If everything went correctly, try to list the files in your Drive using the name of your remote:

```shell
rclone lsd jonasdrive:
```

Now to copy our encrypted tar via rclone, execute:

```shell
rclone copy $HOME/firefly/$(date '+%F').tar.openssl jonasdrive:firefly_backup
```



### Create backup script to do the encryption and sync with rclone

Before we can think of a scheduled timer to execute our backup, we need something to execute at all. Therefore let's create a small bash script that will do the encryption of our `.tar` file and sync the encrypted tar to Google Drive using rclone:

```shell
#!/usr/bin/env bash
set -euo pipefail

echo "### This backups Firefly III database and settings to GDrive"
firefly_home=/home/homepike/firefly
date_hour_min=$(date '+%F-%H-%M')

echo "##### Create backup as .tar"
bash $firefly_home/firefly-iii-backuper.sh backup $firefly_home/$date_hour_min.tar

echo "##### Encrypt .tar using openssl & remove tar locally"
openssl enc -aes-256-cbc -md sha512 -pbkdf2 -iter 1000000 -salt -pass file:$firefly_home/fireflybackup.txt -in $firefly_home/$date_hour_min.tar -out $firefly_home/$date_hour_min.tar.openssl
rm $firefly_home/$date_hour_min.tar

echo "##### Sync encrypted .tar to Google Drive with rclone & remove it locally"
rclone copyto $firefly_home/$date_hour_min.tar.openssl jonasdrive:firefly_backup/$date_hour_min.tar.openssl
rm $firefly_home/$date_hour_min.tar.openssl

echo "##### Retrieve file from Google Drive with rclone again"
rclone copyto jonasdrive:firefly_backup/$date_hour_min.tar.openssl $firefly_home/$date_hour_min.tar.openssl.drive

echo "##### Decrypt encrypted tar from Google Drive to see it would work for restore & remove openssl.drive file"
openssl enc -d -aes-256-cbc -md sha512 -pbkdf2 -iter 1000000 -salt -pass file:$firefly_home/fireflybackup.txt -in $firefly_home/$date_hour_min.tar.openssl.drive -out $firefly_home/$date_hour_min.drive.decrypt.tar
rm $firefly_home/$date_hour_min.tar.openssl.drive

```

The example script also copies the encrypted tar back again from google Drive and decrypts it also again, just to make sure the decryption would also work. It also removes any tmp files used throughout the process and just leaves the final `$firefly_home/$(date '+%F').drive.decrypt.tar` locally for the reference.




## Use systemd Timers instead of cron

[As the docs state we can use a cron job](https://docs.firefly-iii.org/how-to/firefly-iii/advanced/backup/#automated-backup-using-a-bash-script-and-crontab).

But the days where cron was the only alternative are long gone. As the Arch wiki states:

> "...here are many cron implementations, but none of them are installed by default as the base system uses systemd/Timers instead."

See [the benefits section of the Arch wiki](https://wiki.archlinux.org/title/Systemd/Timers#As_a_cron_replacement) also.

So the way to "use cron" today is Systemd Timers: https://wiki.archlinux.org/title/Systemd/Timers

To list all started timers run the following:

```shell
systemctl list-timers
```

See also https://blog.jlcarveth.dev/systemd_timers & https://documentation.suse.com/sle-micro/6.0/html/Micro-systemd-working-with-timers/index.html

In order to create our own systemd based Timer to run our backup, we need 2 additional files:

```shell
fireflybackup.service   - the service that will run the backup script
fireflybackup.timer     - the timer that activates and controls our service
fireflybackup.sh        - our backup script
```

### Create `fireflybackup.service`

Let's first create our `fireflybackup.service` file in `/etc/systemd/system`:

```shell
[Unit]
Description=Service for Firefly III Backup to Google Drive
Wants=fireflybackup.timer

[Service]
Type=oneshot
ExecStart=/home/homepike/firefly/fireflybackup.sh
User=homepike
Group=homepike

[Install]
WantedBy=timers.target
```

> For more information about systemd service files, have a look at https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files

As you may see, I added the `User=` and `Group=` configuration in the `[Service]` section [to have systemd execute my backup script as a user](https://serverfault.com/questions/957323/systemd-start-as-unprivileged-user-in-a-group), that is registered to access rclone or the Gnome Google Drive integration.

Check if your service configuration is correct by running:

```shell
systemd-analyze verify /etc/systemd/system/fireflybackup.*
```

We can now already test drive our systemd powered backup (without the timer) by running:

```shell
sudo systemctl start fireflybackup.service
```

This should already reveal the power of systemd, where we can use the `status` command to look for the actual status of our backup service:

```shell
$ sudo systemctl start fireflybackup.service 
Warning: The unit file, source configuration file or drop-ins of fireflybackup.service changed on disk. Run 'systemctl daemon-reload' to reload units.

$ systemctl status fireflybackup.service
Warning: The unit file, source configuration file or drop-ins of fireflybackup.service changed on disk. Run 'systemctl daemon-reload' to reload units.
○ fireflybackup.service - Service for Firefly III Backup to Google Drive
     Loaded: loaded (/etc/systemd/system/fireflybackup.service; disabled; preset: disabled)
     Active: inactive (dead)

Aug 07 15:25:07 pikeserver fireflybackup.sh[433709]: the input device is not a TTY
Aug 07 15:25:07 pikeserver fireflybackup.sh[433642]: [INFO]  Backing up App & database version numbers.
Aug 07 15:25:07 pikeserver fireflybackup.sh[433642]: [INFO]  Backing up database
Aug 07 15:25:07 pikeserver fireflybackup.sh[433642]: [INFO]  Backing up upload volume
Aug 07 15:25:08 pikeserver fireflybackup.sh[433640]: ##### Encrypt .tar using openssl
Aug 07 15:25:09 pikeserver fireflybackup.sh[433640]: ##### Sync encrypted .tar to Google Drive
Aug 07 15:25:09 pikeserver fireflybackup.sh[433640]: ##### Retrieve file from Google Drive again
Aug 07 15:25:12 pikeserver fireflybackup.sh[433640]: ##### Decrypt encrypted tar from Google Drive to see it would work for restore
Aug 07 15:25:13 pikeserver systemd[1]: fireflybackup.service: Deactivated successfully.
Aug 07 15:25:13 pikeserver systemd[1]: Finished Service for Firefly III Backup to Google Drive.
```

### Create `fireflybackup.timer`

Now that we have the service defined, let's define our `fireflybackup.timer` also in `/etc/systemd/system/`:

```shell
[Unit]
Description=Timer unit for fireflybackup.service
Requires=fireflybackup.service

[Timer]
Unit=fireflybackup.service
OnCalendar=daily

[Install]
WantedBy=timers.target
```

> To configure the `OnCalender` to your time needed, there's also a handy command `systemd-analyze calendar weekly`, `systemd-analyze calendar minutely` etc. See https://wiki.archlinux.org/title/Systemd/Timers#Realtime_timer for more info

Check if your timer configuration is correct by running:

```shell
systemd-analyze verify /etc/systemd/system/fireflybackup.*
```

If that brings no errors, let's start our timer (as we're already used to from systemd services):

```shell
sudo systemctl start fireflybackup.timer
```

You can now check your Google Drive if the encrypted tar was synced correctly.

Since we want to enable our backup to really run regardless of reboots, enable the service as follows:

```shell
sudo systemctl enable fireflybackup.timer
```
