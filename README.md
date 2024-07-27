# firefly-iii-ops
Having a look into personal finance management with Firefly III


Stumbled upon [firefly-iii](https://firefly-iii.org/) while migrating away from Apple products (Macs, iPhones, etc. - see https://github.com/jonashackt/mac-to-linux).

I was looking for a multi-user, OpenSource-based, possibly self-hosted, personal and family finance management solution - which I can use together with my wife.

There are also other alternatives (see https://github.com/awesome-selfhosted/awesome-selfhosted#money-budgeting--management), but Firefly III seemed the most active developed solution to me.

## Install Firefly III with Docker

https://docs.firefly-iii.org/how-to/data-importer/installation/docker/

Download the docker-compose.yml and `.env` files as described. Also change the db passwords and importer url values.

Then run the compose install via:

```shell
docker compose -f docker-compose.yml up -d --pull=always
```


# Find a suitable homeserver

## Why not use a hoster?

I thought about just grabbing some virtual server and run my Docker containers on it. But then I saw the price tags - if you want some RAM (32 gigs), you really need to invest more then 20€. E.g. have a look at Hetzner https://www.hetzner.com/de/cloud where a 32GB machine (shared vCPUs!) start at 38€.

So in only ten month I would have already spend a budget of 380€ - which I could also spend for my own homeserver! Well I wanted to do that since a long time, but didn't find the time to actually do so. Why not start now?

## Thinclients seem to be THE go-to thing today

I remembered bookmarking a Video from c't 3003 https://www.youtube.com/watch?v=cB5n_cWJor8 and started there (German only). They recommended using Thinclients as a start for your homeserver instead of a classic NAS, because they are much cheaper and are available completely without rotating parts like fans or HDDs.

The [Fujitsu Futro S740](https://www.ram-koenig.de/Fujitsu-Futro-S740-ThinClient-Intel-J4105-150GHz-4GB-16GB-SSD-Netzteil) for example is around 80€s as refurbished and [seemed a great starting point](https://www.heise.de/ratgeber/Gebrauchter-Mini-PC-fuer-70-Euro-Thin-Client-Fujitsu-Futro-S740-7485477.html). There's also another recommendation in [this c't 3003 video](https://www.youtube.com/watch?v=K10bMgX0qoc) with the Dell Wyse 5070 Thin Client with Celeron J4105, which is kind of comparable to the Fujitsu model, but available even cheaper at around 60€ already.

I really liked the idea of only spending 60€, but the 4GBs of RAM wouldn't make for a great starting point for my Docker or Kubernetes hosting I guess. Gearing those Celeron J4105 machines up isn't easy, because of quite old hardware standards - and especially the restriction for using only small factor SSDs with only 4cms instead of the usual M.2 SSDs double the size. More than 512GBs would be a great problem to find. Not really great if I perspectively also wanted to host our family photo library and more on the homeserver. So I needed a more up-to-date system!

## More up-to-date architecture

Searching for more capability I found [this reddit thread](https://www.reddit.com/r/HomeServer/comments/1c06m4w/thin_client_recommendations/) where people suggest opting for a [Lenovo ThinkCentre M910q Tiny](https://www.refurbed.de/p/lenovo-thinkcentre-m910q-tiny/71916b/), which brings in much more power with it's Intel Core i5-6500T and a much more modern architecture.  


