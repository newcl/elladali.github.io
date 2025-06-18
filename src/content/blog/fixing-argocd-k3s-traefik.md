---
title: "How I Fixed Argo CD Behind K3s Traefik (No More Redirect Loops!) 🚀"
description: "Avoiding evil annotations and getting Argo CD to work with Traefik in K3s without CRDs."
date: 2025-06-18
author: "Liang Chen"
tags: ["kubernetes", "argocd", "traefik", "k3s", "homelab", "devops"]
---

# How I Fixed Argo CD Behind K3s Traefik (No More Redirect Loops!) 😤➡️😎

Setting up Argo CD behind Traefik in my K3s homelab turned into a rabbit hole of **redirect loops**, **404s**, and Traefik **annotations gone rogue**.

After hours of poking around, here's **exactly what caused the problem** — and how I fixed it. 🔧

---

## 🔥 The Problem

Argo CD was deployed, Traefik (from K3s) was routing traffic, but visiting:

```
https://argo.elladali.com/
```

resulted in:

> ❌ **ERR_TOO_MANY_REDIRECTS**

And sometimes:

> ❌ **404 Not Found**

Even though the Ingress was configured correctly.

---

## 🚩 The Culprits: Evil Annotations

I had these annotations on my Ingress:

```yaml
traefik.ingress.kubernetes.io/forwardedHeaders: "true"
traefik.ingress.kubernetes.io/ssl-redirect: "true"
traefik.ingress.kubernetes.io/protocol: "https"
```

💀 **These caused redirect loops and 404s.** Once I **commented them out**, everything started working.

---

## ✅ The Fix

Here’s what I changed:

### Ingress (🚫 no middleware, no CRDs!)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
    # 🧼 Removed the problematic headers
spec:
  rules:
    - host: argo.elladali.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

### ConfigMap

Make sure your `argocd-cmd-params-cm` ConfigMap looks like this:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  server.insecure: "true"         # allows plain HTTP internally
  server.rootpath: "/"            # important for base routing
  server.basehref: "/"            # same here
  server.enable.proxy: "true"     # tells Argo CD to trust proxy headers
```

Then restart the Argo CD server:

```bash
kubectl -n argocd rollout restart deployment argocd-server
```

---

## 🎉 Result

Now I can access Argo CD through my Cloudflare tunnel + Traefik in K3s, **no TLS in-cluster**, **no Middleware**, and **no CRDs**.

> Clean, simple, and works.

---

If you're getting stuck behind Traefik in a K3s setup, try this minimalist approach before pulling in heavy Helm charts or debugging Traefik's internals.

Happy hacking! 👨‍💻🛰️
