---
title: "Fixing Drone evaluteing secret into environment variable into empty string"
description: "Weird issue secret gets evaluated into empty string by Drone."
date: "2025-06-21"
author: Liang Chen
tags: [drone,ci]

---

## The Subtle Trap of Drone CI Secrets: Why My GITHUB\_TOKEN Was Empty

I recently stumbled upon a frustrating issue in my Drone CI pipeline that took longer to debug than I'd like to admit. The task was simple: after building and pushing new container images, a pipeline step should update the image tags in a Helm chart's `values.yaml` file and commit the change back to the repository. The problem? The `GITHUB_TOKEN` I was using to authenticate with Git was mysteriously evaluating to an empty string, causing the `git clone` command to fail.

### The Problematic Pipeline Step

Here’s a look at the `update-helm-chart` step in my `.drone.yml` file:

```yaml
- name: update-helm-chart
  image: alpine/git
  environment:
    GITHUB_TOKEN:
      from_secret: github_token_for_commit
  commands:
    - apk add --no-cache yq
    - git config --global user.name "Drone CI"
    - git config --global user.email "drone@elladali.com"
    - git clone https://$GITHUB_TOKEN@github.com/newcl/mytube.git /tmp/mytube-repo
    - cd /tmp/mytube-repo
    - git checkout main
    - yq e '.frontend.image.tag = "${DRONE_COMMIT_SHA:0:8}"' -i k8s/mytube/values.yaml
    - yq e '.fastapi.image.tag = "${DRONE_COMMIT_SHA:0:8}"' -i k8s/mytube/values.yaml
    - yq e '.huey.image.tag = "${DRONE_COMMIT_SHA:0:8}"' -i k8s/mytube/values.yaml
    - git add k8s/mytube/values.yaml
    - 'git commit -m "CI: Update mytube images to ${DRONE_COMMIT_SHA:0:8} [skip ci]"'
    - git push origin main
  depends_on:
    - build-frontend
    - build-backend-fastapi
    - build-backend-huey
  when:
    event:
      - push
      - tag
```

The pipeline was correctly configured to pull the `GITHUB_TOKEN` from a secret named `github_token_for_commit`. However, the `git clone` command would fail with an authentication error. When I debugged the pipeline, it became clear that `$GITHUB_TOKEN` was being substituted with an empty string.

### The "Dumb" and Surprising Solution

The baffling part was that other variables in the very same pipeline step were working perfectly. Specifically, `${DRONE_COMMIT_SHA:0:8}` was correctly expanding to the short commit SHA as expected in both the `yq` commands and the commit message. This led me down a rabbit hole of checking my Drone secrets configuration, repository permissions, and everything in between.

After much searching, the solution turned out to be surprisingly simple and, frankly, a bit counterintuitive. The issue was the curly braces `{}` around the variable.

The failing line was:

```bash
- git clone https://${GITHUB_TOKEN}@github.com/newcl/mytube.git /tmp/mytube-repo # This failed
```

The fix was to remove the braces:

```bash
- git clone https://$GITHUB_TOKEN@github.com/newcl/mytube.git /tmp/mytube-repo # This worked
```

That's it. Removing the curly braces allowed Drone to correctly substitute the secret.

This is particularly strange because the official Drone documentation and common shell scripting practices often show `${VARIABLE}` as a valid, and sometimes preferred, way to reference variables to avoid ambiguity. Even more confusing is that `${DRONE_COMMIT_SHA:0:8}` (a built-in Drone variable) *requires* the braces for substring expansion, while the custom secret `$GITHUB_TOKEN` fails when they are present.

This inconsistency in variable handling is, for lack of a better word, a bit dumb. A thread on [Stack Overflow](https://stackoverflow.com/questions/48629789/drone-io-secrets-not-populating-in-yml-appropriately-and-documentation-seems-ina) confirms that this is a known quirk in how Drone handles secrets versus other environment variables.

### Key Takeaway

If you find your secrets are evaluating to empty strings in Drone CI, even when everything seems correctly configured, try this simple fix: **remove the curly braces from your secret variable**.

While `${VARIABLE}` is standard shell syntax and works for Drone's built-in variables, it seems that for secrets injected via `from_secret`, you should use the simpler `$VARIABLE` syntax. It's a subtle distinction, but it can save you hours of debugging. Hopefully, this post saves someone else from the same frustration.