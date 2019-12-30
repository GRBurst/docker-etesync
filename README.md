# Etesync Docker Compose

Docker compose file for an etesync server instance. It is based on this [docker image](https://github.com/GRBurst/docker-etesync-server).

The compose file is ment to run behind a reverse proxy that is available through the `my-reverse-proxy` network.

Remeber to use *more secure* environment variables than the defaults provided here, in this case:

- SUPER_USER: Etesync admin user
- SUPER_PASS: Etesync admin password
- SECRET_KEY (optional): Key for django, otherwise a random one is created by etesync
- MY_UID (optional): user id if you want to access data from the host
- MY_GUID (optional): group id if you want to access data from the host

Note: the uid and guid are only useful if you are using a storage path on the host (not the case here).

Run it with:
```bash
docker-compose pull
docker-compose up -d --remove-orphans --build
```
## Example Setup

### nginx reverse proxy

In many cases, you run it several docker images behind a reverse proxy. Here is how I use it:

Create a `docker network` to let services from different compose files communicate

```bash
docker network create my-reverse-proxy
build: build
```


Nginx `docker-compose` file:

```Dockerfile
version: '3.7'

services:
  nginx:
    container_name: nginx
    build: build
    restart: on-failure
    networks:
      - my-reverse-proxy
      - default
    ports:
      - 80:80
      - 443:443
    volumes:
      - ${TLS_CERTS_DIR:-./.test_certs}:/tls_certs/:ro
      - auth-nginx:/auth/

volumes:
  auth-nginx:

networks:
  my-reverse-proxy:
    name: my-reverse-proxy
```


... and the etesync `docker-compose` file:

```Dockerfile
version: '3'

services:
  etesync:
    container_name: etesync
    image: grburst/etesync:alpine
    restart: always
    networks:
      - my-reverse-proxy
    expose:
      - "3735"
    volumes:
      - data-etesync:/data
    environment:
      SERVER: ${SERVER:-uwsgi}
      SUPER_USER: ${SUPER_USER:-admin}
      SUPER_PASS: ${SUPER_PASS:-admin}

volumes:
  data-etesync:

networks:
  my-reverse-proxy:
    external: true
```


In the nginx configuration, you need to an appropriate `*_pass` to your etesync instance, e.g. a `uwsgi_pass`.

Notes:
1. You can use the hostname `etesync` in the `uwsgi_pass` since they are sharing a network
2. In the example, `https_server.include` is a file providing my default https settings for nginx

It will look similar like the following:

```nginx
server {
    server_name etesync.example.com;
    include conf.d/common/https_server.include;
    client_max_body_size       10m;
    client_body_buffer_size    128k;

    location / {
        include     uwsgi_params;
        uwsgi_pass  etesync:3735;
    }
}
```
