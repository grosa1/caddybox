# Caddy container image

[![Docker Repository on Quay](https://quay.io/repository/joshix/caddy/status "Docker Repository on Quay")](https://quay.io/repository/joshix/caddy)

[![](https://images.microbadger.com/badges/image/joshix/caddy.svg)][microbadger]

This container image encapsulates a [*Caddy*][caddy] HTTP server. It is built `FROM` the [*scratch* image][scratchimg] and executes a single statically-linked binary absent any [*Addon* extensions][caddons]. It includes a tiny `index.html` landing page so that it can be demonstrated without configuration on any Docker host by invoking e.g., `docker run -d -P joshix/caddy`.

The server listens on the container's `EXPOSE`d TCP port #2015 and attempts to fulfill requests with files beneath the container's `/var/www/html/`.

Content should be added by binding over that path with a host volume, or by `COPY`ing/`ADD`ing files there when `docker build`ing an image based on this one. Adding a `Caddyfile` through the same mechanisms allows configuration of the web server and sites as described in the [Caddy documentation][caddydocs].

## Container File System:

* `/bin/caddy` - Server executable
* `/etc/ssl/certs/ca-certificates.crt` - Certificate Authority certificates
* `/var/www/html/` - Server working directory and root of HTTP name space
* `/var/www/html/index.html` - Default landing page

## Adding Content

There are at least two ways to provide Caddy with content and configuration.

* Bind a host file system path over the container's HTTP name space root:

```
$ ls /home/j/site
  index.html
  img/
  [...]
$ docker run -d -p 8080:2015 -v /home/j/site:/var/www/html:ro joshix/caddy
```

OR,

* Build the files into an image based on this one:

```
$ cd /home/j/site.build
$ ls
  Dockerfile
  index.html
  img/logo.png
  [...]
$ cat Dockerfile
  FROM joshix/caddy
  COPY . /var/www/html
$ docker build -t "com.mysite-caddy" .
$ docker run -d -p 8080:2015 com.mysite-caddy
```

## Configuration

To configure Caddy, add `Caddyfile` to the server's working directory:

```
$ ls /home/j/site
  Caddyfile
  index.md
  img/
  [...]
$ cat /home/j/site/Caddyfile
  0.0.0.0:2015
  ext .html .htm .md
  markdown /
  gzip
  [...]
$ docker run -d -p 8080:2015 -v /home/j/site:/var/www/html:ro joshix/caddy
```

### Manual TLS

To serve HTTPS, add certificate and key files, with a Caddyfile naming them:

```
$ ls /home/j/site
  html/
  tls/
$ ls /home/j/site/html
  Caddyfile
  index.html
  img/
  [...]
$ ls /home/j/site/tls
  site.crt
  site.key
$ cat /home/j/site/html/Caddyfile
  0.0.0.0:2015 {
    redir https://site.com # Redirect any HTTP req to HTTPS
  }
  0.0.0.0:2378 {
    tls ../tls/site.crt ../tls/site.key
  }
  [...]
$ docker run -d -p 80:2015 -p 443:2378 -v /home/j/site:/var/www:ro joshix/caddy
```

### Automatic *Let's Encrypt* TLS

Caddy can [automatically acquire and renew TLS keys and certificates][caddyautotls] to secure connections using the *Let's Encrypt* project's ACME protocol.

#### Caddyfile Required

Create a Caddyfile specifying, at minimum, a domain name resolving to the docker host that will arrange for such traffic to be handled by the running caddybox container, and the email address for registration with letsencrypt.

```sh
$ ls /home/j/site
Caddyfile
index.html
$ cat /home/j/site/Caddyfile
lecaddybox.wood-racing.com
  tls j@joshix.com
$ docker run --name com.wood-racing.lecaddybox -d \
-p 80:80 -p 443:443 \
-v /home/j/site:/var/www/html:ro \
-v /home/j/dotcaddy:/.caddy:rw \
joshix/caddy -agree
```

#### Persisting

Certificates, keys, and configuration generated in the letsencrypt exchange are written to files beneath the container’s `/.caddy/letsencrypt/`. The example above arranges for that path to be a Docker *volume*, backing the container's `/.caddy/` with a host directory, `/home/j/dotcaddy/`.

Alternatively, the letsencrypt artifacts can be copied out of the container file system with the `docker cp` command, e.g.:

```sh
$ docker cp com.wood-racing.lecaddybox:/.caddy /backup/dotcaddy
```


[caddons]: https://github.com/mholt/caddy/wiki/Extending-Caddy
[caddy]: https://caddyserver.com
[caddyautotls]: https://caddyserver.com/docs/automatic-https
[caddydocs]: https://caddyserver.com/docs
[microbadger]: https://microbadger.com/images/joshix/caddy "Get your own image badge on microbadger.com"
[scratchimg]: https://hub.docker.com/_/scratch/
