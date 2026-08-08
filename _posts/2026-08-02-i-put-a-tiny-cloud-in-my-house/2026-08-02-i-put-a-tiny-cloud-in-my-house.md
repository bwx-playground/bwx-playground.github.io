---
layout: post
title:  "I Put a Tiny Cloud in My House (Part 1 of 3)"
date: 2026-08-02 06:00:00 +0700
categories: kubernetes homelab
tags: k3s raspberry_pi devops
---

I have a Raspberry Pi 4 sitting on a shelf with 8GB of RAM, which is roughly the same amount of memory my first laptop had. On it, I run something that sounds a bit ridiculous for a $80 board: a full Kubernetes cluster, with its own private app store, its own encrypted internal network, and a robot that deploys my apps for me while I sleep.

This is the first of three posts about why I built it, how the automation works, and the surprisingly chaotic story of deploying my first real app onto it. Let's start with why.

## Why bother?

At work, "the cluster" is something huge, shared, and slightly scary to touch. If I break something there, it's a real problem. I wanted a version of that world I could break on purpose, learn from, and rebuild in twenty minutes if I really messed it up. A Raspberry Pi is perfect for that: cheap enough that breaking it doesn't hurt, small enough that every shortcut and every corner cut becomes obvious quickly.

So the whole project has one rule baked into it: favor simplicity, and always make it easy to tear down and rebuild from scratch. Nothing here is trying to be "production grade" — it's trying to be a safe place to practice for production grade.

## What's actually running on it

k3s is Kubernetes, just trimmed down to fit on small hardware — same ideas, much lighter footprint. Once that's up, though, an empty cluster doesn't do much on its own. Here's the cast of characters I added around it, in plain terms:

**The front door (Gateway API).** Every request into the cluster, whether it's coming from my own laptop or from the internet, walks through one of two "front doors." One faces my home network only. The other faces the outside world. Nothing gets in any other way.

**The bodyguard (Istio, in "ambient" mode).** Every bit of traffic between two things running in the cluster gets escorted and encrypted, automatically, without either side having to know it's happening. I didn't have to write any code for this — it happens at the network level, quietly, in the background.

**The private app store (zot).** Before I can run my own apps, I need somewhere to store the container images and the deployment templates I build. Instead of pushing everything to a public registry, I run my own tiny one, reachable only from my home network.

**The ID badge office (cert-manager).** Internal-only services still deserve real HTTPS, not a browser warning. This piece automatically requests and renews genuine certificates for everything running inside the house.

**The phone line to the internet (a Cloudflare tunnel).** I didn't want to open any ports on my home router — that's usually where trouble starts. Instead, a small process inside the cluster opens an outbound connection to Cloudflare, and that becomes the only path in from the public internet.

**The locked box (Sealed Secrets).** Passwords and tokens need to live in Git alongside everything else, or automation can't fully work. Sealed Secrets lets me encrypt a credential so that only my own cluster can ever decrypt it — safe to commit, safe to look at.

Here's roughly how those pieces sit next to each other:

![architecture overview](/assets/img/architecture-overview.svg)

Two doors in, one shared encrypted mesh on the inside, a handful of specialists (registry, certificates, secrets) doing one job each. Nothing in that picture is doing anything clever on its own — the interesting part is how little *I* have to do once it's wired up this way.

## A small taste of the config

I promised myself I wouldn't turn this into a manual, but one piece is short enough to be worth actually showing: attaching a new service to a "front door" is just a Gateway API `HTTPRoute`, and it's almost embarrassingly small.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: get-public-ip
spec:
  parentRefs:
    - name: raspi-gateway        # the external front door
  hostnames:
    - "ip.icipicip.com"
  rules:
    - backendRefs:
        - name: get-public-ip
          port: 80
```

That's genuinely the whole routing decision: which door, which hostname, which service behind it. Everything else — the TLS, the mTLS between pods, the encryption — is handled elsewhere, automatically, because of how the pieces above are wired together.

Every one of those pieces exists to answer one question: once this cluster exists, how do I actually *use* it, without doing everything by hand every single time?

That question is where things get interesting — and it's the whole subject of Part 2: teaching the cluster to deploy my apps by itself.
