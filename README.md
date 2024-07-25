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
