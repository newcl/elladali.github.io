---
title: "Fixing Argo CD 404 Behind Traefik and Cloudflare Tunnel"
description: "How I debugged a 404 error when exposing Argo CD with Traefik Ingress and Cloudflare Tunnel, and what caused it."
date: "2025-06-17"
author: Liang Chen
tags: [k3s, argocd, traefik, ingress, homelab]
layout: ../../layouts/PostLayout.astro
---

When exposing [Argo CD](https://argo-cd.readthedocs.io/) in a homelab using Traefik as the Ingress controller and Cloudflare Tunnel for HTTPS, I encountered a persistent `404 Not Found` on the root path (`/`). Here's a breakdown of what caused it, how I verified my routing setup, and what fixed the problem.

---

## ✅ The Setup

I used the following technologies:

- **K3s** as my Kubernetes distribution.
- **Traefik** as the built-in Ingress controller.
- **Cloudflare Tunnel** to expose `https://argo.elladali.com` to the public.
- **Argo CD** installed in the `argocd` namespace.

My Ingress looked like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
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
          - path: /fallback
            pathType: Prefix
            backend:
              service:
                name: fallback-service
                port:
                  number: 80
```

The `/fallback` route was added to verify if traffic was even reaching my cluster — and it worked! A 404 on `/fallback/` confirmed that my **DNS**, **Cloudflare Tunnel**, and **Ingress routing** were functioning correctly.

---

## ❌ The Problem: `404 Not Found` on `/`

Despite routing being correct, `https://argo.elladali.com/` kept returning a 404.

I had customized the Argo CD config with this ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  server.insecure: "true"
  server.rootpath: "/"
  server.basehref: "/"
```

At first glance, setting `server.rootpath` and `server.basehref` to `/` seems harmless. But it turns out this breaks things.

---

## 🧠 The Root Cause

According to [Argo CD documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#serving-argo-cd-from-a-subpath), `server.rootpath` is only needed **when Argo CD is hosted at a sub-path like `/argocd`**.

Setting it to `/` tells the Argo CD UI to treat **every request** as if it were under a sub-path — which it's not. As a result, all requests to `/`, `/login`, `/api`, etc., get routed incorrectly and return 404.

> This is a common mistake in reverse proxy setups. You don't need to set `rootpath` if you're using a clean domain like `argo.elladali.com/`.

---

## ✅ The Fix

1. **Remove the path settings from the ConfigMap:**

   ```yaml
   data:
     server.insecure: "true"
     # Remove or comment out:
     # server.rootpath: "/"
     # server.basehref: "/"
   ```

2. **Apply the changes and restart the Argo CD server:**

   ```bash
   kubectl apply -f argocd-cmd-params-cm.yaml
   kubectl -n argocd rollout restart deployment argocd-server
   ```

3. **Confirm it's working:**

   Either port-forward internally:

   ```bash
   kubectl -n argocd port-forward svc/argocd-server 8080:80
   curl -I http://localhost:8080/  # Should return 302 → /login
   ```

   Or refresh `https://argo.elladali.com/` in your browser — the UI should load!

---

## ✅ Bonus: Using `/fallback` to Confirm Ingress

Creating a dummy route like `/fallback` is a simple trick to check if your Traefik Ingress is working.

Here’s how to create a basic fallback service:

```bash
kubectl create deployment fallback --image=nginx -n argocd
kubectl expose deployment fallback --port=80 --target-port=80 -n argocd --name=fallback-service
```

Now visit `https://argo.elladali.com/fallback/`. If it shows a 404 from Nginx or a default page, you know the Ingress works.

---

## 🎉 Summary

If you’re exposing Argo CD on a root domain (`argo.example.com`) behind Traefik:

- **✅ Don’t set `server.rootpath` or `server.basehref`** unless you’re using a sub-path.
- **✅ Use a `/fallback` path to test Ingress behavior.**
- **✅ Confirm everything with `curl` or port-forward before debugging Cloudflare or TLS.**

This setup now runs cleanly in my homelab, with Argo CD securely exposed via Cloudflare Tunnel — no TLS management, no headaches.

---

Got a similar setup or issue? Let me know — happy to dig deeper.
