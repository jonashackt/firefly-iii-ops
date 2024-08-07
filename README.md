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


## Automatically Backup to Cloud Provider using cron & Google Drive

Backing up your database is great, but you should also place the backups on another machine to mitigate hardware failure. 

If you want to use your Google Drive, you can use [this description here to connect your Gnome Desktop to your Drive](https://github.com/jonashackt/mac-to-linux/blob/main/README.md#google-drive-desktop-file-sync).


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


### Sync encrypted tar to Google Drive with cron

[As the docs state we can now use a cron job](https://docs.firefly-iii.org/how-to/firefly-iii/advanced/backup/#automated-backup-using-a-bash-script-and-crontab).

Now to create a cron job to run daily, [simply create a file `fireflybackup` at `/etc/cron.daily`](https://askubuntu.com/a/2369/451114) with the following contents:

> THE FILE SHOULDN'T USE the `.sh` filename ending - [if it does, cron doesn't execute the script!](https://askubuntu.com/a/416479/451114)

```shell
#!/usr/bin/env bash
set -euo pipefail

echo "### This backups Firefly III database and settings to GDrive"

echo "##### Create backup as .tar"
bash $HOME/firefly/firefly-iii-backuper.sh backup $HOME/firefly/$(date '+%F').tar

echo "##### Encrypt .tar using openssl"
openssl enc -aes-256-cbc -md sha512 -pbkdf2 -iter 1000000 -salt -pass file:fireflybackup.txt -in $HOME/firefly/$(date '+%F').tar -out $HOME/firefly/$(date '+%F').tar.openssl

echo "##### Sync encrypted .tar to Google Drive"
cp $HOME/firefly/$(date '+%F').tar.openssl "/run/user/1000/gvfs/google-drive:host=googlemail.com,user=your.user/upoeiutpoewutpoewutpoi/öalkjsfölöjah797asggsdga"

echo "##### Retrieve file from Google Drive again"
cp "/run/user/1000/gvfs/google-drive:host=googlemail.com,user=your.user/upoeiutpoewutpoewutpoi/öalkjsfölöjah797asggsdga/$(date '+%F').tar.openssl" $HOME/firefly/$(date '+%F').tar.openssl.drive

echo "##### Decrypt encrypted tar from Google Drive to see it would work for restore"
openssl enc -d -aes-256-cbc -md sha512 -pbkdf2 -iter 1000000 -salt -pass file:fireflybackup.txt -in $HOME/firefly/$(date '+%F').tar.openssl.drive -out $HOME/firefly/$(date '+%F')-drive-decrypt.tar
```

Be sure to change your Google Drive folder location from `google-drive:host=googlemail.com,user=your.user/upoeiutpoewutpoewutpoi/öalkjsfölöjah797asggsdga` to the value of your's (you can retrieve the value of your backup folder in your Google Drive by navigating to the dir inside your GNOME files explorer and click onto the three dots, then on `copy location`). 


Be also sure to also make it executable via `chmod +x` - and move the password file also into `/etc/cron.daily`

```shell
chmod +x /etc/cron.daily/fireflybackup
sudo mv fireflybackup.txt /etc/cron.daily
```

Now in order [to testdrive our cron job](https://askubuntu.com/a/416479/451114), let's execute it like cron would do:

```shell
cd /etc/cron.daily
run-parts --verbose /etc/cron.daily
```

The example script also copies the encrypted tar back again from google Drive and decrypts it also again, just to make sure the decryption would also work.


#### cp: cannot stat '/run/user/1000/gvfs/google-drive Permission denied with root

A cron job is executed with the root user, thus it's root trying to access our Google Drive - which leads to the following error:

```shell
#### Sync encrypted .tar to Google Drive
cp: cannot stat '/run/user/1000/gvfs/google-drive:host=googlemail.com,user=jonas.hecht/0ACdy8c0ajOxBUk9PVA/1cpLn14MZOCjrsz0yYreYIAYQGhhOYbgr': Permission denied
run-parts: /etc/cron.daily/fireflybackup exited with return code 1
```

It [seems to be an issue with in-memory mounts of type fuse.portal](https://unix.stackexchange.com/a/762694/140406), which the GNOME Desktop uses with its online services:

> "Apparently /run/user/<$UID>/gvfs is an addition by the GNOME desktop environment specifically for services associated with running GNOME desktop."

So we need a solution to run our cron job from our user instead of root. As [this answer states we need to switch from using `cron.daily` dir to the crontab](https://serverfault.com/a/352837/326340). Therefore without sudo run:

```shell
crontab -e
```

This opens your user's cron tab and the needed format for defined jobs is:

```shell
minute hour day-of-month month day-of-week command
```

So in my case this is:

```shell
1 12 * * * homepike /home/homepike/firefly/fireflybackup
```


## Use systemd Timers instead of cron

The days where cron was the only alternative are long gone. As the Arch wiki states:

> "...here are many cron implementations, but none of them are installed by default as the base system uses systemd/Timers instead."

But we might also get into other problems using systemd Timers at the end. Why not look for a general purpose Cloud Storage Syncing solution?


## Using rclone

I already stumbled upon it, but didn't have the time to look into it. But it might be THE tool for the job https://github.com/rclone/rclone

Let's try it out. Install on Manjaro via:

```shell
pamac install rclone
```

Directly afterwards we can configure it with:

```shell
rclone config
```

In order to configure Google Drive with rclone, just have a look into the docs: https://rclone.org/drive/


So the way today is Systemd Timers: https://wiki.archlinux.org/title/Systemd/Timers

