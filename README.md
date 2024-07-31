# firefly-iii-ops
Having a look into personal finance management with Firefly III


Stumbled upon [firefly-iii](https://firefly-iii.org/) while migrating away from Apple products (Macs, iPhones, etc. - see https://github.com/jonashackt/mac-to-linux).

I was looking for a multi-user, OpenSource-based, possibly self-hosted, personal and family finance management solution - which I can use together with my wife.

There are also other alternatives (see https://github.com/awesome-selfhosted/awesome-selfhosted#money-budgeting--management), but Firefly III seemed the most active developed solution to me.

> If you need some hints on how to find a suitable homeserver, have a look at this https://github.com/jonashackt/homeserver


## Install Firefly III with Docker

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




