---
title: "Homelab Initiation"
date: 2026-10-26T16:31:12-04:00
author: "UnknownSilicon"
cover: ""
tags: ["homelab", "tech"]
keywords: ["", ""]
description: ""
showFullContent: false
readingTime: false
hideComments: false
table-of-contents: false
toc-auto-numbering: false

draft: false
---

# The Hook

In early 2026, I started to acquire bits of used server hardware for cheap, and sometimes free. A homelab has been on my mind for a while; I've been wanting to make a media server, network backups, self hosted applications, and a space to practice miscellaneous network administration.

I eventually hit the critical point: 2 Dell Poweredge R320s, 2 old QNAP NASes, and a whole slew of hard drives to go with it. The only problem now was networking. The one spot in my apartment with enough open power outlets was away from the rest of my networking gear.

Luckily, there was a coax cable there. Unluckily, the coax splitter in the basement was tightened with the torque of... I'm not sure how to finish this joke but it was very tight and on a loose cable that made it near impossible to unscrew for quite some time.

Thankfully, having cool strong tall women as friends has many benefits, including being able to unscrew the coax cable.

<!-- I would put a picture of this cable situation if I had one -->

# The Build

With the splitter fixed, I was able to properly use a MoCA adapter. I finally have actual network access. I throw some hard drives into one of the servers, hook up the network and power, plugin in a VGA monitor and keyboard, and boot into an Ubuntu Server installation drive.

And with that, Xenon was born.

{{< img class=center-fig src=/Servers.jpg width=500px alt="The servers in a stack in a corner of a room. From bottom to top, there's an empty QNAP NAS, an empty Dell Poweredge R320, an R320 with all drive bays full, powered on, with a sticker 'Xenon - 192.168.1.', and a full QNAP NAS on top. On top of the server stack, there is a laptop and VGA monitor.">}}

## Media Server
For the start, I've focused on the media server. The software stack is centered around a slew of Docker containers and Tailscale for external access. A docker compose file contains configurations for the server components:
- DIUN
    - Docker Image Update Notifier - Sends me notifications weekly when there's updates available for docker containers
- Nginx Proxy
    - Provides external hostname based routing, rate limiting, and SSL for all services
- Jellyfin
    - Media server. This is pretty close to a stock installation
- Sonarr and Radarr
    - Manage TV Shows and Movies, respectively. Keeps track of releases, scores media with quality profiles
- Clonarr
    - Automatically update and configure quality profiles for Sonarr and Radarr to match TRaSH-Guides recommended configurations, with a few tweaks
- Seerr
    - Provides an interface for users to request new media and forward tracking to Sonarr and Radarr
- Bazarr
    - Automatically find and update subtitles for content. Dubiously useful, but it's here
- Tdarr
    - Distributed media transcoder. Allows using my desktop GPU and CPU to transcode media
    - I may write a whole separate post about this and the nuances of transcoding and getting direct streaming working in as many places as possible

For security, all of these services are only accessible on the local network, with Tailscale for external access. Tailscale certainly hasn't been the most performant for streaming externally, with ocassional freezing, but this is something I've yet to investigate further. I'll include some snippets of my compose file at the end of this post, but the actual configs aren't public for several reasons. If you're interested though, feel free to reach out :3

{{< img class=center-fig src="/Server Infra.png" width=700px alt="Network diagram. There is a large blue box behind many items labeled 'Docker'. Outside of the box, there is the local network, internet, tailscale, and data storage. Inside, there are the services listed above as well as a box labeled 'Other Stuff'. DIUN is connected to the internet directly. The NGINX proxy is connected to the local network and tailscale, as well as incoming connections from Jellyfin, Bazarr, Clonarr, Sonarr, Seerr, Radarr, and Tdarr. Bazarr, Clonarr, and Seerr are all connected to both Sonarr and Radarr. Sonarr, Radarr, Jellyfin, and Tdarr are connected to Data Storage. Tdarr is also connected to the local network with a label 'Media Transcode'.">}}

Over the past ~6 months of operation, I've done a Lot of tweaking of various configurations in attempts to improve tracking, ensure accuracy of quality profiles for media, improve streaming quality, improve ease of maintenance, and limit file sizes. A large part of these adjustments came in the form of Tdarr automations to ensure uniform files and widespread compatibility for direct streaming. In future posts, I'll talk about my Tdarr automations, potentially other stuff I'm hosting, and maybe even some adjcent Home Automation stuff I'm working on :3

{{< img class=center-fig src=/jelly.png width=700px alt="Screenshot of the Jellyfin admin dashboard, showing the server name and version and file paths.">}}

{{< code title="Docker Compose" >}}
```yaml
  tdarr:
    container_name: tdarr
    image: ghcr.io/haveagitgat/tdarr:latest
    restart: unless-stopped
    ports:
      - 8266:8266 # server port
    environment:
      - TZ=America/New_York
      - PUID=1000
      - PGID=1000
      - UMASK_SET=002
      - serverIP=0.0.0.0
      - serverPort=8266
      - webUIPort=8265
      - internalNode=false
      - inContainer=true
      - ffmpegVersion=7
      - nodeName=Xenon
      - auth=false
      - openBrowser=false
      - maxLogSizeMB=10
      - cronPluginUpdate=
    volumes:
      - /data/tdarr/server:/app/server
      - ./config/tdarr/configs:/app/configs
      - /data/tdarr/logs:/app/logs
      - /data/media:/media
      - /data/media/transcode_cache:/temp
    devices:
      - /dev/dri:/dev/dri

  seerr:
    image: ghcr.io/seerr-team/seerr:v3.2.0
    init: true
    container_name: seerr
    environment:
      - LOG_LEVEL=warning
      - TZ=America/New_York
      - PORT=5055
    volumes:
      - ./config/seerr:/app/config
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:5055/api/v1/settings/public || exit 1
      start_period: 20s
      timeout: 3s
      interval: 15s
      retries: 3
    restart: unless-stopped

  radarr:
    container_name: radarr
    image: ghcr.io/hotio/radarr
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=002
      - TZ=America/New_York
      - WEBUI_PORTS=7878/tcp
    volumes:
      - ./config/radarr:/config
      - /data/:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:7878/login"]
      interval: 15s
      timeout: 5s
      retries: 3

  sonarr:
    container_name: sonarr
    image: ghcr.io/hotio/sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - UMASK=002
      - TZ=America/New_York
      - WEBUI_PORTS=8989/tcp
    volumes:
      - ./config/sonarr:/config
      - /data/:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8989/login"]
      interval: 15s
      timeout: 5s
      retries: 3

  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    user: 1000:1000
    ports:
      - 8096:8096/tcp
      - 7359:7359/udp
    volumes:
      - ./config/jellyfin:/config
      - /data/jellyfin/cache:/cache
      - type: bind
        source: /data/media
        target: /media
    restart: 'unless-stopped'
    environment:
      - JELLYFIN_PublishedServerUrl=<REDACTED>
    extra_hosts:
      - 'host.docker.internal:host-gateway'

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - ./config/bazarr:/config
      - /data:/data
    restart: unless-stopped
    depends_on:
      - radarr
      - sonarr

  proxy:
    image: nginx
    volumes:
    - type: bind
      source: ./config/nginx.conf
      target: /etc/nginx/conf.d/default.conf
      read_only: true
    - type: bind
      source: <REDACTED>
      target: /etc/nginx/ssl/fullchain.crt
      read_only: true
    - type: bind
      source: <REDACTED>
      target: /etc/nginx/ssl/privkey.key
      read_only: true
    ports:
      - 80:80
      - 443:443
    restart: unless-stopped
    depends_on:
      - jellyfin
    container_name: proxy

  clonarr:
    image: ghcr.io/prophetse7en/clonarr:v2.5.3
    container_name: clonarr
    restart: unless-stopped
    environment:
      - TZ=America/New_York
      - PUID=1000
      - PGID=1000
      - TRUSTED_PROXIES=proxy
    volumes:
      - ./config/clonarr:/config
    depends_on:
      radarr:
        condition: service_healthy
      sonarr:
        condition: service_healthy

  diun:
    image: crazymax/diun:latest
    command: serve
    volumes:
      - "./config/diun/data:/data"
      - "./config/diun/diun.yml:/diun.yml"
      - "/var/run/docker.sock:/var/run/docker.sock"
    environment:
      - "TZ=America/New_York"
      - "LOG_JSON=false"
      - "DIUN_WATCH_WORKERS=20"
      - "DIUN_WATCH_SCHEDULE=0 0 * * 6"
      - "DIUN_WATCH_JITTER=30s"
      - "DIUN_PROVIDERS_DOCKER=true"
      - "DIUN_PROVIDERS_DOCKER_WATCHBYDEFAULT=true"
    labels:
      - "diun.enable=true"
    restart: always
```
{{< /code >}}