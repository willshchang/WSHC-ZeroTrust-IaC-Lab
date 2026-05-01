# Tailscale IaC — keys.tf

**Document Type:** IaC Reference  
**Author:** Will Chang, Customer Success Engineer  
**Audience:** IT Administrator / Tailscale CSE Reference  
**Last Updated:** April 2026  
**Repository:** https://github.com/willshchang/WSHC-ZeroTrust-IaC-Lab  
**Official Reference:** https://tailscale.com/kb/1085/auth-keys

---

## Overview

`keys.tf` generates pre-authentication keys via Terraform — allowing 
devices to join the Tailnet without manual browser login.

---

## Auth Key vs Tailscale SSH — Two Different Things

A common misconception is that Tailscale SSH and auth keys are 
related. They are completely separate:

| Feature | Purpose | When used |
|---|---|---|
| **Auth key** | Device enrollment — joins Tailnet | One time — bootstrap |
| **Tailscale SSH** | Ongoing SSH access — replaces passwords | Every SSH session |

**Auth key** = the employee badge that gets you in the building  
**Tailscale SSH** = your ID card that controls which rooms you access

---

## Why Auth Keys Matter for IaC

**Without auth key — manual enrollment:**
```bash
ssh admin@vm
sudo tailscale up
# Opens browser URL → human must authenticate manually
# Cannot be scripted or automated at scale
```

**With auth key — fully automated:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up \
  --authkey=$(terraform output -raw vm_auth_key) \
  --ssh \
  --accept-routes \
  --advertise-exit-node
# No browser, no human, VM joins Tailnet in seconds
```

---

## Key Design

```hcl
resource "tailscale_tailnet_key" "vm_auth_key" {
  reusable      = false   # single-use — limits blast radius
  ephemeral     = false   # VM persists after going offline
  preauthorized = true    # skips manual approval in admin console
  expiry        = 3600    # expires in 1 hour — use immediately
  description   = "Tailscale auth key for tinyco-vm"
  tags          = [var.tag_server]
  depends_on    = [tailscale_acl.policy]
}
```

**`reusable = false`** — single-use key. If leaked, attacker can 
only enroll one device before the key is consumed.

**`expiry = 3600`** — 1 hour expiry. Use immediately after 
`terraform apply`. Expired keys cannot be used — generate a new 
one by running `terraform apply` again.

---

## The `tag:terraform` Ownership Chain

**The problem:**
The OAuth client cannot generate auth keys for `tag:server` unless 
it owns that tag — but giving the OAuth client direct ownership of 
`tag:server` creates a security concern.

**The solution — manager tag pattern:**
OAuth client → assigned tag:terraform (admin console)
tag:terraform → owns tag:server (acl.tf tagOwners)
tag:terraform → owns tag:subnet-router (acl.tf tagOwners)

The OAuth client owns `tag:terraform`. `tag:terraform` owns 
`tag:server`. This chain gives Terraform permission to generate 
auth keys tagged as `tag:server` without direct ownership.

> **Reference:** Known Tailscale OAuth limitation documented in:  
> https://github.com/tailscale/tailscale/issues/8299  
> https://github.com/tailscale/tailscale/issues/15456

---

## Retrieving the Key

```bash
# After terraform apply
terraform output -raw vm_auth_key
```

Key value starts with `tskey-auth-...`

**Security — secret sprawl prevention:**
The output is marked `sensitive = true` — never printed in plain 
text during `terraform plan` or `terraform apply`. Always retrieve 
via `terraform output -raw` and pipe directly — never store in 
plain text files or shell history.

**Production pattern:**
```bash
# Pipe directly without storing
sudo tailscale up --authkey=$(terraform output -raw vm_auth_key) --ssh
```

---

## Production Expansion

```hcl
# CI/CD ephemeral node — auto-removed when offline
resource "tailscale_tailnet_key" "ci_key" {
  reusable      = true
  ephemeral     = true    # removed from Tailnet after going offline
  preauthorized = true
  expiry        = 3600
  description   = "CI/CD ephemeral key"
  tags          = [var.tag_server]
}

# Separate keys per environment
resource "tailscale_tailnet_key" "prod_key" {
  description = "Production server key"
  tags        = ["tag:prod-server"]
}

resource "tailscale_tailnet_key" "staging_key" {
  description = "Staging server key"
  tags        = ["tag:staging-server"]
}
```

---

## Official References

| Topic | URL |
|---|---|
| Auth keys | https://tailscale.com/kb/1085/auth-keys |
| Tailnet key resource | https://registry.terraform.io/providers/tailscale/tailscale/latest/docs/resources/tailnet_key |
| OAuth tag ownership | https://github.com/tailscale/tailscale/issues/8299 |