# Docker Compose with NGINX proxy and Letsencrypt automatic SSL

- [Purpose](#purpose)
- [Usage](#usage)

## Purpose

Provide two ways to add new websites/APIs with automatic SSL support and renewal:

1. Simply update the list of domains inside `.env` file. No need to setup nginx, proxies, mess with docker-compose.yml file, or configuring new containers (see [Usage Out of The Box](#a-usage-out-of-the-box)).

2. Spin up new docker-compose.yml instance (see [WordPress example](https://github.com/evertramos/wordpress-docker-letsencrypt)).

## Usage

### A. Usage Out of The Box

Simply clone this repo, create and update `.env` file and run docker-compose (docker must be installed first):
```bash
git clone https://github.com/ecoinomist/docker-letsencrypt-nginx-proxy.git webproxy
cd webproxy
cp .env.sample .env # then update WEBSITES_DOMAINS list
docker network create webproxy
docker-compose up -d
```

Each website listed at `WEBSITES_DOMAINS` is expected to have `index.html` file at this dynamic location `<WEBSITES_PATH>/<domain>.<tld>/dist/index.html`.
The `/dist` part can be changed inside `websites/conf.d/default.conf` by updating `root` path.

### B. Usage with Custom Configurations

Follow these steps:

1. Copy the content of `docker-compose.yml`, as of below:

```bash
version: '3'
services:
  nginx:
    image: nginx
    labels:
        com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy: "true"
    container_name: nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ${NGINX_FILES_PATH}/conf.d:/etc/nginx/conf.d
      - ${NGINX_FILES_PATH}/vhost.d:/etc/nginx/vhost.d
      - ${NGINX_FILES_PATH}/html:/usr/share/nginx/html
      - ${NGINX_FILES_PATH}/certs:/etc/nginx/certs:ro

  nginx-gen:
    image: jwilder/docker-gen
    command: -notify-sighup nginx -watch -wait 5s:30s /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
    container_name: nginx-gen
    restart: always
    volumes:
      - ${NGINX_FILES_PATH}/conf.d:/etc/nginx/conf.d
      - ${NGINX_FILES_PATH}/vhost.d:/etc/nginx/vhost.d
      - ${NGINX_FILES_PATH}/html:/usr/share/nginx/html
      - ${NGINX_FILES_PATH}/certs:/etc/nginx/certs:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro
      - ./nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro

  nginx-letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: nginx-letsencrypt
    restart: always
    volumes:
      - ${NGINX_FILES_PATH}/conf.d:/etc/nginx/conf.d
      - ${NGINX_FILES_PATH}/vhost.d:/etc/nginx/vhost.d
      - ${NGINX_FILES_PATH}/html:/usr/share/nginx/html
      - ${NGINX_FILES_PATH}/certs:/etc/nginx/certs:rw
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      NGINX_DOCKER_GEN_CONTAINER: "nginx-gen"
      NGINX_PROXY_CONTAINER: "nginx"
```

2. Create an `.env` file and say where you will locate the nginx files:

```
NGINX_FILES_PATH=./nginx
```

3. Change the file `docker-compose.yml` with your own settings:

3.1. Set your PROXY Network

Your website/API container must be in the same network of your nginx proxy.
```bash
networks:
  default:
    external:
      name: your-network-name
```

3.2. Set your IP address (optional)

On the line `ports` add as follow:
```bash
    ports:
      - "YOUR_PUBLIC_IP:80:80"
      - "YOUR_PUBLIC_IP:443:443"

```

4. Get the latest version of **nginx.tmpl** file (only if you have not cloned this repostiry)

```bash
curl https://raw.githubusercontent.com/jwilder/nginx-proxy/master/nginx.tmpl > nginx.tmpl
```
Make sure you are in the same folder of docker-compose file, if not, you must update the the settings `- ./nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro`.

5. Start your project
```bash
docker-compose up -d
```

> Please note that when running a new container to generate certificates with LetsEncrypt it may take a few minutes, depending on multiples circunstances.


Your proxy is ready to go!

### If you want to test how it works please check this working sample (docker-compose.yml)

[wordpress-docker-letsencrypt](https://github.com/evertramos/wordpress-docker-letsencrypt)

Or you can run your own containers with the option `-e VIRTUAL_HOST=foo.bar.com` alongside with `LETSENCRYPT_HOST=foo.bar.com`, exposing port 80 and 443, and your certificate will be generated and always valid.


## Credits

All credits goes to:
- [@jwilder](https://github.com/jwilder/nginx-proxy)
- [@JrCs](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)


### Special thanks to:

- [@buchdag](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/pull/226#event-1145800062)
- [@fracz](https://github.com/fracz) - Update repo to use `.env` file
