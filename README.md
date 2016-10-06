# docker-proxy

Transparent caching proxy server for Docker containers, run in a Docker
container. It can speed up the dependency-fetching part of your application
build process.

This container has been modified to support cache_peers. (another proxy to forward proxy requests).

It has also been changed in the sense that squid ports (e.g. 3129) are now exposed. The reason for this is, that the approach used in the project we forked from, to route to the internal container running squid via ip rule / ip route / iptables rules doesn´t seem to work anymore in newer Docker versions, due to the DOCKER_ISOLATION chain, which prevents containers running in different docker networks from communicating with each other.
Therefore we recommend using a DNAT rule.

iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 172.24.0.80:3129

## Instructions for Use

Modify docker-compose.yml and set the environment variables.

add a DNAT rule to forward HTTP traffic originating from the container to port 3129. Dockers Masquerading in this case (POSTROUTING) will not kick in first.

```
sudo docker build -t docker-proxy .
```

Then run with:

```
docker-compose up docker-proxy
```
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to 172.24.0.80:3129
Your other Docker containers will automatically use
the proxy, whether or not they were already running.

NOTE: This project is _not_ designed to be run with a simple `docker run` - it
requires you set an additional DNAT rule to be run on the docker host. There might be a way of using --net:host to tun on the host directly.

## Overview

`docker-compose` will fire up a Docker container running Squid.

If you want to see Squid in operation, you can (in another terminal) attach
to the `docker-proxy` container - it is tailing the access log, so will show a
record of requests made.

## HTTPS Support

The proxy server supports HTTPS caching via Squid's [SSL Bump] feature. To
enable it, start with moifying docker-compose.yml to expose port 3130:

```
docker-compose up docker-proxy
```

The server will decrypt traffic from the server and encrypt it again using its
own root certificate. HTTPS connections from your other Docker containers will
fail until you install the root certificate.

To test HTTPS support, do this in another console after starting the proxy:

```
cd test
sudo docker build -t test-proxy .
sudo docker run --rm test-proxy
# Should print "All tests passed"
```

[SSL Bump]: http://wiki.squid-cache.org/Features/SslBump
[`detect-proxy.sh`]: test/detect-proxy.sh
[`test/Dockerfile`]: test/Dockerfile

## Notes

This proxy configuration is intended to be used in situation, where you are on an intranet Docker host and your containers need a working http connection to e.g. install updates during run.
