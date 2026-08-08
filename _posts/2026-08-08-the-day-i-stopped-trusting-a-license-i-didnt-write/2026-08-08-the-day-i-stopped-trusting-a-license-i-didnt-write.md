---
layout: post
title:  "The Day I Stopped Trusting a License I Didn't Write"
date: 2026-08-08 09:00:00 +0700
categories: kubernetes homelab
tags: terraform opentofu iac
---

Back in [Part 2]({% post_url 2026-08-03-teaching-my-cluster-to-deploy-itself %}) I described how every ArgoCD Application on this cluster gets declared in Terraform — one `module` block per app, `terraform plan` before anything ships, no `kubectl apply` typed from memory. It's a small setup: one state file, one module, no environments to juggle. It's also been sitting there quietly for weeks doing exactly what it's supposed to. Nothing about it was broken.

I switched it to OpenTofu anyway. Here's the reasoning, for anyone else wondering if the fork is worth the trouble.

## What OpenTofu actually is

In August 2023, HashiCorp changed Terraform's license from MPL 2.0 — real, OSI-approved open source — to the Business Source License (BUSL) 1.1, a license HashiCorp itself owns and can rewrite the terms of on every future release. In response, a group of companies forked the last MPL-licensed codebase and handed it to the Linux Foundation. It's since joined the CNCF too, with Oracle, Spacelift, Gruntwork, and others backing it instead of one vendor.

Mechanically, it's the same tool wearing a different name: same HCL, same provider plugin protocol, same state file format, same `init`/`plan`/`apply` verbs. Point it at existing state and it just reads it. No conversion step, no relearning.

## No, using Terraform isn't "wrong"

I want to be precise about this, because it's easy to hear "BUSL" and assume something's being violated. It isn't. HashiCorp's restriction is narrow: you can't take Terraform and resell it as a *competing hosted product* — that clause was aimed at companies building paid platforms on top of free Terraform, not at someone running `terraform apply` against their own Raspberry Pi. For this repo, BUSL changes nothing about what I'm allowed to do today.

What it changes is what I can *trust about tomorrow*. HashiCorp already rewrote the deal once, with no vote, after a decade of MPL. HashiCorp is now owned by IBM. Nothing says the terms hold still from here — and unlike an OpenTofu governed by multiple companies under the Linux Foundation, there's no structural reason they'd have to.

## The part that made it more than a principle

If it were only a license argument, I'd probably have left `argocd-apps/` alone — it's a small, low-stakes repo, not worth ceremony. What tipped it was that OpenTofu has actually kept moving since the fork, shipping things the free Terraform CLI still doesn't have: native state encryption, `for_each` over provider configurations, and — the one that made me grin — **OCI registry support for providers and modules**. I already run `zot` as an OCI registry for Helm charts (that's [Part 1]({% post_url 2026-08-02-i-put-a-tiny-cloud-in-my-house %})'s "private app store"). The same registry could, in principle, serve provider binaries too. I'm not doing that today, but it's a nice door to have standing open.

## What actually changed, and what deliberately didn't

Given the near-zero switching cost, the move itself was almost boring:

```
brew install opentofu
tofu init -backend-config=backend.hcl
tofu plan
```

Same R2 backend, same `argoproj-labs/argocd` provider, same state — `tofu plan` came back clean against what `terraform plan` already showed. No migration, just a different binary reading the same files.

The more interesting part was figuring out what to *leave alone*. It would've been easy to go rename everything with "terraform" in it, and that would have been wrong. OpenTofu deliberately kept several literal names unchanged for compatibility:

- The `terraform { }` configuration block is still spelled `terraform`, not `tofu` — that's an HCL keyword, and OpenTofu didn't rename it.
- `terraform.tfvars` is still the exact filename OpenTofu auto-loads. Renaming it would just mean nothing gets loaded.
- The state key I'd already chosen, `argocd-apps/terraform.tfstate`, stays put — it's a literal object path in R2, and "renaming" it would mean copying live state to a new key, not a docs edit.

What *did* change was purely the human-facing layer: `README.md` and `CLAUDE.md` now say `tofu init` / `tofu plan` / `tofu apply` instead of the `terraform` equivalents, and describe the tool as OpenTofu. The `.terraform.lock.hcl` picked up OpenTofu's own hash entries on next init, committed like any other lockfile bump.

## What I'd tell past me

Don't switch tools because a blog post told you to. Switch when the switching cost is close enough to zero that the only real question left is "which set of future incentives do I want to depend on" — and for a homelab repo with one state file and one module, that was true here almost by default. The interesting lesson wasn't OpenTofu itself; it was noticing how much of "migrating" a tool is actually just careful *not*-changing — knowing which names are load-bearing compatibility guarantees versus which ones are just prose that happened to say "Terraform."
