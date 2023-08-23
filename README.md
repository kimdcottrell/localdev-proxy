# A reverse proxy, powered by Traefik, designed for Dockerized local development environments

Sometimes, you want to use domain names for your local development containers. 

## Starting this reverse proxy and accessing the admin panel

Firstoff, if you're running any webserver directly on your local machine, shut it down. It will cause port conflict errors.

With that done, add these domains to `/etc/hosts`. This may require `sudo`:

```
127.0.0.1   admin.traefik
127.0.0.1   whoami.traefik
```

Now you'e good to go. Run:

`docker compose up --build` (possibly with `-d` if you want to run this in the background so you can still use that terminal window)

And if things are working correctly, you should:

1. Be able to hit `http://admin.traefik` in your browser and see the Traefik dashboard
2. Be able to visit `http://whoami.traefik` in your browser (or curl for this one) and see an echoserver response

## How to use with your application's local development docker-compose.yml

Add in your desired domain name to your `/etc/hosts`. This may require `sudo`. For example:

`127.0.0.1	something.local.dev`

In your OTHER `docker-compose.yml` file, where your application stack lives, edit things so the traefik labels are included, but alter `apilocaldev` to whatever you want it to be, and alter `traefik.http.services.apilocaldev.loadbalancer.server.port` to be the `EXPOSE`'d port inside your `Dockerfile` for that container's image. Make sure you also include the `networks` value and the reference to this container stack's externally-presenting network so your application stack can access it. 

```
services:
  api:
    ...
    labels:
      # Explicitly tell Traefik to expose this container
      - traefik.enable=true
      # The domain the service will respond to
      - traefik.http.routers.apilocaldev.rule=Host(`api.local.dev`)
      # Allow request only from the predefined entry point named "web"
      - traefik.http.routers.apilocaldev.entrypoints=web # this is working with the port 80 entrypoint in the traefik config (a different docker-compose.yml)
      # Tell Traefik to use the port 8001 to connect to `protoinvestapi`
      - traefik.http.services.apilocaldev.loadbalancer.server.port=8001 # this can be anything, but mirror the change back to the Dockerfile
    networks:
      - proxy
networks:
  proxy:
    external: true
```

And if all goes well, you should be able to visit `http://app.local.dev` or whatever your domain name was in your browser instead of the usual `http://localhost:1337` or whatever port you picked. 

# Relevant blog articles 

[How to use Traefik Proxy without exposing the Docker socket (HTTP Filter Edition)](https://kimdcottrell.com/posts/how-to-use-traefik-proxy-without-docker-socket-exposed-http-filter-edition/)

# Changes soon to come!

Self-signed Certs! This will allow for https-enabled localdev environments. e.g. `https://app.local.dev`