---

title: "How I Securely Exposed Argo CD Without Managing TLS"
description: "Using Cloudflare Tunnel and Traefik to expose Argo CD with HTTPS, while running everything inside my K3s homelab on plain HTTP."
pubDate: 2025-06-17
author: "Liang Chen"
--------------------

If you're running Argo CD (or anything else) in a homelab and want to expose it securely over the internet without touching TLS certs or opening ports, this post is for you. I'll walk you through how I exposed Argo CD using Cloudflare Tunnel, Traefik, and K3s—**without managing HTTPS myself**.

---

## Goal

* Argo CD should be accessible at `https://argo.elladali.com`
* TLS should be fully handled without me dealing with certs
* Everything inside the cluster stays on plain HTTP
* No need to open any ports to the public

Turns out this is pretty easy with **Cloudflare Tunnel** and **Traefik's iptables redirection** in K3s.

---

## Setup Overview

* K3s installed with default Traefik ingress
* Argo CD running with `--insecure` (i.e., on plain HTTP)
* `cloudflared` tunnel runs **on the host**, outside K3s
* DNS managed by Cloudflare, domain is `argo.elladali.com`

And here's the `cloudflared` config I use:

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

Note the key bit: I'm forwarding traffic to `http://localhost:80`. You’d expect that to go nowhere, but since **K3s modifies iptables**, this is actually routed into Traefik inside the cluster.

---

## Request Flow Explained

Here’s what happens when I open Argo CD in a browser:

### 1. Browser → Cloudflare

I visit `https://argo.elladali.com`. Cloudflare serves as the edge, and since it's in **SSL Full mode**, it expects HTTPS from both the browser and origin.

### 2. Cloudflare → `cloudflared`

Cloudflare forwards the request **over HTTPS** to the `cloudflared` tunnel client running on my homelab machine. The tunnel connection is TLS-encrypted.

### 3. `cloudflared` → localhost:80

Here’s the magic: `cloudflared` decrypts the HTTPS and forwards the raw HTTP request to `http://localhost:80`. That’s where K3s’s iptables kicks in...

### 4. iptables → Traefik

K3s modifies iptables so that traffic hitting `localhost:80` is redirected to the in-cluster Traefik ingress controller.

### 5. Traefik → Argo CD

Traefik sees the incoming HTTP request, matches it against the Ingress for `argo.elladali.com`, and forwards it to the `argocd-server` Kubernetes service. Argo CD is running with `--insecure`, so it's perfectly happy with HTTP.

Everything works.

---

## TLS Termination Point

Here’s the full picture:

```text
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

TLS is terminated at **Cloudflare**. From there on, it’s all HTTP, which is fine because it stays inside my trusted LAN/K3s environment.

Even Argo CD’s login and GitHub OAuth work without issue, because from the browser’s perspective, everything is happening over HTTPS.

---

## Why This Works So Well

* No need for cert-manager or messing with TLS
* Secure access over the internet via Cloudflare
* Internal traffic stays simple (plain HTTP)
* Works with any number of apps (`drone.elladali.com`, etc.)

I didn’t expect this setup to be so smooth, but it turned out rock solid.

If you're running a homelab and want to expose services like Argo CD, Drone, or Gitea without opening ports or worrying about HTTPS, this approach is hard to beat.

---

## Next Steps

I’ll probably containerize `cloudflared` and run it in-cluster next, just to clean things up. But as-is, this setup is production-grade for personal projects.

Let me know if you're doing something similar or want help setting it up.
