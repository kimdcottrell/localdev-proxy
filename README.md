# HTTPS for Docker localdevs

Sometimes, you want to use domain names for your local development containers. And you want them to be TLS-enabled. And you want to run many webservers across different projects at a time, all with domain names and no ports.

Here is how to do it.

## DO THIS FIRST

Install this: https://github.com/FiloSottile/mkcert 

Run this:

```
cd certs
mkcert -install
mkcert -key-file wildcard-local-dev-key.pem -cert-file wildcard-local-dev-cert.pem *.local.dev
```

## Important notes

For every webserver container you intend to proxy back to this, add this to `/etc/hosts`:

```
# tab delimit this, and change out the "my-site-name-here" with whatever you want
127.0.0.1    my-site-name-here.local.dev
```

Sadly, `/etc/hosts` does not allow for wildcards. If you want wildcards so you never have to do this again, check out `dnsmasq`.

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

In your OTHER `docker-compose.yml` file, where your application stack lives, edit things so the traefik labels are included, but alter `webserverlocaldev` to whatever you want it to be, and alter `traefik.http.services.webserverlocaldev.loadbalancer.server.port` to be the `EXPOSE`'d port inside your `Dockerfile` for that container's image. Make sure you also include the `networks` value and the reference to this container stack's externally-presenting network so your application stack can access it. 

```
# Changeable substrings: `webserverlocaldev`, `webserver`, `webserver.local.dev`, `8001`
services:
  webserver:
    ...
    networks:
      - default
      - proxy
    ...
    labels:
      # Explicitly tell Traefik to expose this container
      - traefik.enable=true
      # Tell Traefik you are planning a redirection, and to include the needed middleware
      - traefik.http.middlewares.webserverlocaldev-redirect-web-secure.redirectscheme.scheme=https
      - traefik.http.routers.webserverlocaldev.middlewares=webserverlocaldev-redirect-web-secure
      # The domain the service will respond to, and what is in your /etc/hosts
      - traefik.http.routers.webserverlocaldev-web.rule=Host(`webserver.local.dev`)
      # Allow request only from the predefined entry point named "web"
      - traefik.http.routers.webserverlocaldev-web.entrypoints=web # this is working with the port 80 entrypoint in the traefik config (a different docker-compose.yml)
      # Let's redirect!
      - traefik.http.routers.webserverlocaldev-web-secure.rule=Host(`webserver.local.dev`)
      - traefik.http.routers.webserverlocaldev-web-secure.tls=true
      - traefik.http.routers.webserverlocaldev-web-secure.entrypoints=web-secure
      # What is essentially in this container's Dockerfile or image's Dockerfile under the `EXPOSE` setting
      - traefik.http.services.webserverlocaldev-web-secure.loadbalancer.server.port=80 # this can be anything, but mirror the change back to the Dockerfile via EXPOSE

networks:
  # Creating our own network allows us to connect between containers using their service name.
  default:
    driver: bridge
  proxy:
    external: true
```

And if all goes well, you should be able to visit `https://app.local.dev` or whatever your domain name was in your browser instead of the usual `http://localhost:1337` or whatever port you picked. 

An example of this in action is available here: https://github.com/kimdcottrell/wordpress-localdev-template/blob/main/docker-compose.yml 

# Relevant blog articles 

[How to use Traefik Proxy without exposing the Docker socket (HTTP Filter Edition)](https://kimdcottrell.com/posts/how-to-use-traefik-proxy-without-docker-socket-exposed-http-filter-edition/)

[5 simple steps to achieving https and domain names for Docker localdev environments](https://kimdcottrell.com/posts/5-steps-to-achieving-https-and-domain-names-for-docker-local-development-envs/)