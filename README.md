# Nick's dockerized homelab / NAS / media server


## About
I run a lot of services on my homelab servers, managing them and keeping them up to date has become significantly easier since I've decided to dockerize everything.

Having this docker-compose file also makes this setup portable. In fact I have recently moved everything to a synology box and it's been working great.

I figured I'd publish this as a good starting point for anyone who wants to run a similar setup.

![Docker containers preview](preview.png?raw=true "Docker containers preview")

## List of services:
- sonarr, radarr, lidarr: to keep track of series, movies, music and queue downloads.
- sabnzbd, deluge: to facilitate the downloads.
- plex server: to pull metadata and server all the media.
- unifi-controller: to manage my unifi access points.

I also run [pihole](https://pi-hole.net/) (DNS level adblocking) and [unbound](https://www.nlnetlabs.nl/projects/unbound/about/) (validating, recursive, caching DNS resolver) but have moved those to a raspberry pi, config [here](https://github.com/petalas/rpi).


## To update all images and restart containers as needed:
```
sudo docker-compose pull
sudo docker-compose up -d
```

## If you find any of my projects useful, please consider making a small donation:
|    | Address |
-----|-----
BTC  | 3B1gtQhVQj1x5c1jzVHHzq2eGKyEqf7rQb
ETH  | 0x18Df0b2C16d3E3927Ef74e02268fE36949bb6b2c
ADA  | addr1vyqvaqs56d90ha55duh04eh59y7fcps8duplgm987nyhv9qp3n05f
USDT | 0x96A37296B30Bf19A917c34810b3f040192Be2e67