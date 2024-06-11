# Auto-proxy - Docker mod for SWAG

This mod gives SWAG the ability to auto-detect running containers, across multiple hosts, via labels and automatically enable reverse proxy for them.

## Requirements:
- This mod needs the [universal-docker mod](https://github.com/linuxserver/docker-mods/tree/universal-docker) installed and set up with either mapping `docker.sock` or setting the environment variable `DOCKER_HOST=remoteaddress`.
- Other containers detected via `docker.sock` to be auto-detected and reverse proxied should be in the same [user defined bridge network](https://docs.linuxserver.io/general/swag#docker-networking) as SWAG.
- Containers to be auto-detected and reverse proxied must have a label `swag=enable` at a minimum.
- To benefit from curated preset proxy confs we provide, the container name must match the container names that are suggested in our readme examples (ie. `radarr` and not `Radarr-4K`).

## Labels:
- `swag=enable` - required for auto-detection
- `swag_address=containername` - *optional* - overrides upstream app address. Can be set to an IP or a DNS hostname. Defaults to `container name`.
- `swag_port=80` - *optional* - overrides *internal* exposed port (if no preset conf and this label not set, auto-proxy will default to first detected exposed port)
- `swag_proto=http` - *optional* - overrides internal proto (defaults to http)
- `swag_url=containername.domain.com` - *optional* - overrides *server_name* (defaults to `containername.*`)
- `swag_auth=authelia` - *optional* - enables auth methods (options are `authelia`, `authentik`, `ldap` and `http` for basic http auth)
- `swag_auth_bypass=/api,/othersubfolder` - *optional* - bypasses auth for selected subfolders. Comma separated, no spaces.


In SWAG docker arguments, set an environment variable `DOCKER_MODS=linuxserver/mods:universal-docker|linuxserver/mods:swag-auto-proxy` and either add a volume mapping for `/var/run/docker.sock:/var/run/docker.sock:ro`, or set an environment var `DOCKER_HOST=remoteaddress`.

## Security Consideration:
Mapping the `docker.sock`, especially in a publicly accessible container is a security liability. Since this mod only needs read-only access to the docker api, the recommended method is to proxy the `docker.sock` via a solution like [tecnativa/docker-socket-proxy](https://hub.docker.com/r/tecnativa/docker-socket-proxy), limit the access, and set `DOCKER_HOST=` to point to the proxy address.

Here's a sample compose yaml snippet for tecnativa/docker-socket-proxy:
```yaml
  dockerproxy:
    image: ghcr.io/tecnativa/docker-socket-proxy:latest
    container_name: dockerproxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped
    environment:
      - CONTAINERS=1
      - POST=0
```
Then the env var in SWAG can be set as `DOCKER_HOST=dockerproxy`. This will allow docker cli in SWAG to be able to retrieve info on other containers, but it won't be allowed to spin up new containers.

## Multiple Hosts:

If both `DOCKER_HOST` and `docker.sock` volumes are provided this mod will detect containers using both connections. As noted in the [requirements](#requirements), containers detected via `docker.sock` must be in the same user defined network or have `swag_address` label set.

Multiple remote hosts can be used via `DOCKER_HOST` by separating hosts with a comma. Additionally, a friendly host name can be assigned by suffixing the host with `|friendlyName`. If the host is not assigned a friendly name it will have a default generated like `hostN` where N is the position of the host in `DOCKER_HOST`. Example:

```
DOCKER_HOST=192.168.0.100:2375|serverA,192.168.0.110:2375|serverB,192.168.0.130:2375
```

* Host: `192.168.0.100:2375` -- Friendly Name: `serverA`
* Host: `192.168.0.110:2375` -- Friendly Name: `serverB`
* Host: `192.168.0.130:2375` -- Friendly Name: `host3`

If the detected containers for each host do not have the `swag_address` label set then the host IP will be used.

If the detected containers for each host do not have the `swag_url` label set then the default can be configured with friendly host name inserted into the url as either a suffix or prefix using the env `HOST_INSERT`. Example:

* container name: `swag`
* `DOCKER_HOST=192.168.0.100:2375|serverA`
* `HOST_INSERT`
  * (unset) => nginx `server_name swag.*`
  * `prefix` => nginx `server_name serverA-swag.*`
  * `suffix` => nginx `server_name swag-serverA.*`