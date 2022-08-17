---
title: "Integrate Appwrite into existing Traefik Deployment"
categories:
  - Blog
  - Self-Hosted
tags:
  - Appwrite
  - docker-compose
  - traefik
---

Just a few days ago I decided to test Appwrite for a personal project of mine. Normally seeing an existing docker-compose file for deployment is a very good sign for an easy and speedy test.

This will just be some snippets. The full files are available in [my github repository](https://github.com/santschi/hosting-snippets/blob/main/dev/appwrite/docker-compose.yml)

## The Situation

Turns out that the standard configuration for Appwrite requires a deployment of Traefik that mounts its configuration and certificates from Appwrite.

The official docker-compose file can be found [here](https://appwrite.io/install/compose).

```yaml
services:
  traefik:
    image: traefik:2.7
    ...
    command:
      - --providers.file.directory=/storage/config
    ...
    volumes:
     ...
      - appwrite-config:/storage/config:ro
      - appwrite-certificates:/storage/certificates:ro
  ...
  appwrite:
   ...
    volumes:
    ...
      - appwrite-config:/storage/config:rw
      - appwrite-certificates:/storage/certificates:rw
```

Appwrite itself takes care of the configuration of traefik (at least the provider) and the creation and maintenance of the certificate(s).

## My Problem

This approach seems great if you are deploying Appwrite by itself on a (virtual) machine. But if there are other services running on the same node giving Appwrite this much control over the reverse-proxy seems like a major security risk. In my mind Appwrite is less security critical than Traefik. So I wanted a setup where Appwrite does not in any way access Traefik configuration.

## Integrate Appwrite into existing Traefik instance

First I removed the traefik service from the file.

```yaml
appwrite:
    image: appwrite/appwrite:0.15.3
    user: "${UID}:${GID}"
    container_name: appwrite
    restart: unless-stopped
    networks:
      - appwrite
      - appwrite-gateway
    labels:
      # Expose this container via Traefik
      - traefik.enable=true
      # Tell Traefik which network to use
      - traefik.docker.network=appwrite-gateway
      # Define the appwrite service to use later
      - traefik.http.services.appwrite_api.loadbalancer.server.port=80
      # What Entrypoint to use (defined in the traefik deployment) Port 80 in my case
      - traefik.http.routers.appwrite_api_http.entrypoints=web
      # Filter for the expected url
      - traefik.http.routers.appwrite_api_http.rule=(Host(`api.example.ch`) && PathPrefix(`/`))
      # Forward requests to the previously defined service
      - traefik.http.routers.appwrite_api_http.service=appwrite_api
      # Workaround for global https redirect you probably do not need this.
      - traefik.http.routers.appwrite_api_http.priority=2000
      # https
      - traefik.http.routers.appwrite_api_https.entrypoints=web-secure
      - traefik.http.routers.appwrite_api_https.rule=(Host(`api.example.ch`) && PathPrefix(`/`))
      - traefik.http.routers.appwrite_api_https.service=appwrite_api
      - traefik.http.routers.appwrite_api_https.tls=true
      - traefik.http.routers.appwrite_api_https.tls.certresolver=cloudflare-resolver
      - traefik.http.routers.appwrite_api_https.tls.domains[0].main=*.example.ch


  appwrite-realtime:
    image: appwrite/appwrite:0.15.3
    user: "${UID}:${GID}"
    entrypoint: realtime
    container_name: appwrite-realtime
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=appwrite-gateway"
      - "traefik.http.services.appwrite_realtime.loadbalancer.server.port=80"
      - traefik.http.routers.appwrite_realtime.rule=(Host(`api.example.ch`) && PathPrefix(`/v1/realtime`))
      - traefik.http.routers.appwrite_realtime.service=appwrite_realtime
      #Workaround for global https redirect
      - traefik.http.routers.appwrite_realtime.priority=2000
      # wss
      - traefik.http.routers.appwrite_realtime_wss.entrypoints=web-secure
      - traefik.http.routers.appwrite_realtime_wss.rule=(Host(`api.example.ch`) && PathPrefix(`/v1/realtime`))
      - traefik.http.routers.appwrite_realtime_wss.service=appwrite_realtime
      - traefik.http.routers.appwrite_realtime_wss.tls=true
      - traefik.http.routers.appwrite_realtime_wss.tls.certresolver=cloudflare-resolver
      - traefik.http.routers.appwrite_realtime_wss.tls.domains[0].main=*.example.ch
    networks:
      - appwrite
      - appwrite-gateway

networks:
  ...
  appwrite-gateway:
    external: true
    name: appwrite-gateway
```

I added a dedicated network for communication between Appwrite and Traefik called `appwrite-gateway`. This is created in my traefik deployment.

```yaml
networks:
  appwrite-gateway:
    driver: bridge
    name: appwrite-gateway
```

Don't forget to add the `.env` file that is [provided by Appwrite](https://appwrite.io/install/env).

The [documentation](https://appwrite.io/docs/installation) is very good for most other use-cases.

## Alternative Solution: Proxy Chain

Since the prepared compose file has configured traefik with `constraint-labels` it should be possible to chain the `appwrite-traefik` behind another traefik instance. To do this properly you would need to add constraint labels to all your containers, otherwise your main instance would not ignore the appwrite container labels as needed.

I decided not to use this approach because:

- It would need a lot of reconfiguration of the existing infrastructure on my server
- Unnecessary duplication of work by using two proxies chained.
- I like a clear separation of proxy and service.
- It makes running Appwrite as a different user possible (or at least a whole lot easier)
