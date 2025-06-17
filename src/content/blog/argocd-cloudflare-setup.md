---
title: "How I Securely Exposed Argo CD Without Managing TLS"
description: "Using Cloudflare Tunnel and Traefik to expose Argo CD with HTTPS, while running everything inside my K3s homelab on plain HTTP."
date: "2025-06-17"
draft: false
author: "Liang Chen"
tags:
  - k3s
  - cloudflare
  - argocd
  - traefik
---

## Goal

- Argo CD should be accessible at `https://argo.elladali.com`  
- TLS should be fully handled without me dealing with certs  
- Everything inside the cluster stays on plain HTTP  
- No need to open any ports to the public  

Turns out this is pretty easy with **Cloudflare Tunnel** and **Traefik's iptables redirection** in K3s.

---

## Setup Overview

- K3s installed with default Traefik ingress  
- Argo CD running with `--insecure` (i.e., on plain HTTP)  
- `cloudflared` tunnel runs **on the host**, outside K3s  
- DNS managed by Cloudflare, domain is `argo.elladali.com`  

Here's the `cloudflared` config:

```yaml
# /etc/cloudflared/config.yml
tunnel: myk3s-tunnel
credentials-file: /etc/cloudflared/<your-uuid>.json

ingress:
  - hostname: argo.elladali.com
    service: http://localhost:80
  - hostname: drone.elladali.com
    service: http://localhost:80
  - service: http_status:404
```

---

## Request Flow Explained

Here’s what happens when I open Argo CD in a browser:

1. Browser → Cloudflare  
Browser visits https://argo.elladali.com. Cloudflare serves as the edge and handles TLS in SSL Full mode.

2. Cloudflare → cloudflared  
The request is tunneled over HTTPS to cloudflared on my homelab machine.

3. cloudflared → localhost:80  
Cloudflared decrypts HTTPS and forwards it to plain HTTP localhost:80.

4. iptables → Traefik  
K3s modifies iptables so that traffic hitting localhost:80 is routed to Traefik.

5. Traefik → Argo CD  
Traefik forwards the HTTP request to Argo CD, which is running with --insecure.

---

## TLS Termination Point

```
[BROWSER]
   ↓ HTTPS
[CLOUDFLARE EDGE]
   ↓ HTTPS
[CLOUDFLARED]
   ↓ HTTP
[IPTABLES REDIRECT]
   ↓ HTTP
[TRAEFIK IN K3S]
   ↓ HTTP
[ARGO CD SERVICE]
```

TLS ends at Cloudflare. Everything inside stays HTTP, safely in your LAN.

---

## Why This Works So Well

- No cert-manager or TLS headaches  
- Secure external access via Cloudflare  
- Simple internal HTTP stack  
- Extendable to other apps (e.g., Drone, Gitea)

---

## Next Steps

I’ll containerize cloudflared next and run it inside K3s, just for tidiness. But this setup already works great.
