+++
title = "Enabling Brotli content encoding on CapRover"
date = "2025-05-16"
tags = ["CapRover", "Docker", "NGINX"]
+++

[CapRover](https://caprover.com/) is a fantastic piece of software that I came across last year, it solves a problem
I've had with Docker web app deployment for a long time. It's an all-in-one solution for building and deploying
containerised applications, and I've had a really good experience with it.

As a part of its architecture, it provisions a single NGINX container in front of all of your apps and automatically
configures your apps as upstream backends. You're not really supposed to mess with this container too much, although you
can modify the NGINX configuration file via the UI.

[Brotli](https://github.com/google/brotli) is a compression algorithm designed as an improvement to gzip to reduce the
bandwith used by web apps and to reduce the time it takes to decompress HTTP responses.

NGINX does support Brotli encoding, but requires an additional module to be installed before it can be enabled. Because
CapRover deploys NGINX as a Docker container it's not easily possible to add these modules. The 'Docker' way is to build
a new image which extends the existing NGINX image, add these modules, and use that container instead.

Luckily fholzer maintains a well-updated [NGINX-Brotli image](https://github.com/fholzer/docker-nginx-brotli) with
everything already set up as we require. The image can be downloaded from Dockerhub:

`docker pull fholzer/nginx-brotli:v1.28.0`

CapRover also provides a mechanism to override the NGINX image used by CapRover. To maximise compatability I would
recommend using the closest NGINX version to the one your current caprover install uses:

```
docker ps --format '{{.Image}} {{.Names}}' | grep captain-nginx | awk '{split($1, a, ":"); print a[2]}'
> 1.27.2
```

Also, please note that future CapRover updates may break this, and you may need to use an updated fholzer/nginx-brotli
image in the future.

Now we have to set this define our new NGINX image as an override in the Captain config-override.json file.

`/captain/data/config-override.json`

Here is how mine looks

`{"nginxImageName": "fholzer/nginx-brotli:v1.28.0"}`

Once saved, you can then restart Captain and a new NGINX container from our nginx-brotli image will load. Everything
should still work as normal at this point, if it doesn't then you need to stop and figure out what went wrong before
proceeding to the next steps.

`docker service update captain-captain --force`

The next step can be achieved in the Caprover UI by navigating to Settings -> NGINX Configurations ->
Base Config Location in nginx container. Or by editing `/captain/generated/nginx/nginx.conf`. At the very bottom of the
file, before `include /etc/nginx/conf.d/*.conf;` you want to add the following lines:

```
brotli on;
brotli_static on;
brotli_min_length 100;
brotli_buffers 16 8k;
brotli_comp_level 4;
brotli_types application/javascript
    application/x-javascript
    application/json
    application/xml
    application/rss+xml
    application/atom+xml
    application/vnd.ms-fontobject
    application/x-font-ttf
    application/x-font-opentype
    application/font-woff
    application/font-woff2
    font/ttf
    font/otf
    font/woff
    font/woff2
    image/svg+xml
    text/css
    text/plain
    text/xml
    text/x-component
    text/javascript;
```

Then either in the UI press 'Save and Update' or restart Caprover:

`docker service update captain-captain --force`

Brotli should now be working on your website. You can verify using tools such as [KeyCDN's Brotli Test](https://tools.keycdn.com/brotli-test).