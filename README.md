# scalelite-run

A simple way to deploy Scalelite as for production using docker-compose.
This project is a fork of jfederico's [scalelite-run](https://github.com/jfederico/scalelite-run)

## Overview

[Scalelite](https://github.com/blindsidenetworks/scalelite) is an open-source load balancer, designed specifically for [BigBlueButton](https://bigbluebutton.org/), that evenly spreads the meeting load over a pool of BigBlueButton servers. It makes the pool of BigBlueButton servers appear to a front-end application such as Moodle [2], as a single and yet very scalable BigBlueButton server.

It was released by [Blindside Networks](https://blindsidenetworks.com/) under the AGPL license on March 13, 2020, in response to the high demand of Universities looking into scaling BigBlueButton in response to the [COVID-19 pandemic lock-downs](https://campustechnology.com/articles/2020/03/03/coronavirus-pushes-online-learning-forward.aspx).

The full source code is available on GitHub and pre-built docker images can be found on [DockerHub](https://hub.docker.com/r/blindsidenetwks/scalelite).

Scaleite itself is a ruby on rails application.

For its deployment it is required some experience with BigBlueButton and Scalelite itself, and all the tools and components used as part of the stack such as redis, postgres, nginx, docker and docker-compose, as well as ubuntu and AWS infrastructure.

For those new to system administration or any of the components mentioned the article [Scalelite lazy deployment
](https://jffederico.medium.com/scalelite-lazy-deployment-745a7be849f6) is a step-by-step guide on how to complete a full installation of Scalelite on AWS using this script. Also [Scalelite lazy deployment (Part II)](https://jffederico.medium.com/scalelite-lazy-deployment-part-ii-ca3e4bf82f8d) is a step-by-step guide to complete the installation with support for recordings.

## Installation (short version)

On an Linux machine available to the Internet (AWS EC2 instance, LXC container, VMWare machine etc).

### Prerequisites

This machine needs to be updated and have installed:

- Git
- [Docker](https://docs.docker.com/engine/install/)
- [Docker Compose](https://docs.docker.com/engine/install/)

### Fetching the scripts

```
git clone https://github.com/michelleDeko/scalelite-run
cd scalelite-run
```

### Initializing environment variables

Create a new .env file based on the dotenv file included.

```
cp dotenv .env
```

Most required variables are pre-set by default, the ones that must be set before starting are:

```
SECRET_KEY_BASE=
LOADBALANCER_SECRET=
SL_HOST=
DOMAIN_NAME=
```

Obtain the value for SECRET_KEY_BASE and LOADBALANCER_SECRET with:

```
sed -i "s/SECRET_KEY_BASE=.*/SECRET_KEY_BASE=$(openssl rand -hex 64)/" .env
sed -i "s/LOADBALANCER_SECRET=.*/LOADBALANCER_SECRET=$(openssl rand -hex 24)/" .env
```

Set the hostname on SL_HOST (E.g. sl)

```
sed -i "s/SL_HOST=.*/SL_HOST=sl" .env
```

Set the domain name on DOMAIN_NAME (E.g. example.com)

```
sed -i "s/DOMAIN_NAME=.*/DOMAIN_NAME=example.com" .env
```

Start the services.

```
docker-compose up -d
```

Now, the scalelite server is running, but it is not quite yet ready. The database must be initialized.

```
docker exec -i scalelite-api bundle exec rake db:setup
```

The BBB servers must be added.

```
docker exec -i scalelite-api bundle exec rake servers:add[https://bbb25.example.com/bigbluebutton/api,secret]
docker exec -i scalelite-api bundle exec rake servers:enable[bbb25.example.com]
```

## Generate LetsEncrypt SSL certificates manually

Depending on if you want to use Multitenancy or not this can be done differently.
If you only need one domain (e.g. sl.example.com) you can use the `init-letsencrypt.sh` script, that's included in this repo

If you want to setup SSL with Multitenancy, you will need a wildcard certificate. In this documentation I will use DNS challenge with Cloudflare.
For this, first get a API token from Cloudflare: [https://dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens)
Click "Create token" and use "Edit zone DNS" as a template. Under "Zone Resources" you can now choose your domain.

After you got your API token, create a file the file `/etc/letsencrypt/cloudflare.ini` with the following:
```
# Cloudflare API token used by Certbot
dns_cloudflare_api_token = your-secure-token
```
Set the permissions: `chmod 600 /etc/letsencrypt/cloudflare.ini`
Now we can use that file to create the wildcard certificate:

```
certbot certonly -d *.sl.example.com --dns-cloudflare --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini --dns-cloudflare-propagation-seconds 60
```

