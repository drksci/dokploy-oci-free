# drksci Dokploy Recipe — public control plane + home Swarm workers

An intentionally un-fragile setup: a small always-on **control plane** in the
cloud, and **compute that lives on your own gear** at home. Free.

```
                 ┌──────────────────────────────┐
   GitHub push → │  OCI Always-Free A1 VM        │  public IP, always on
   Let's Encrypt │  Dokploy manager + Traefik    │  → panel, ingress, webhooks
   webhooks  →   │  (Swarm manager)              │
                 └───────────────┬──────────────┘
                                 │  Tailscale mesh (WireGuard)
              ┌──────────────────┼───────────────────┐
              │                  │                   │
      ┌───────┴──────┐   ┌───────┴──────┐   ┌────────┴───────┐
      │ home-imac-01 │   │ bare-metal   │   │ any Docker/    │
      │ Colima VM    │   │ Linux box    │   │ Colima host    │
      │ Swarm worker │   │ Swarm worker │   │ Swarm worker   │
      └──────────────┘   └──────────────┘   └────────────────┘
```

## Why this shape isn't fragile

- **Control plane is public + always-on** (OCI free A1). GitHub webhooks reach
  it, Let's Encrypt validates, the panel is always up — none of the
  tailnet-only reachability problems.
- **All inter-node traffic rides Tailscale**, not the public internet. Swarm's
  control (2377) and overlay (7946/4789) go over WireGuard, so workers join
  from behind home NAT with **no port-forwarding, no exposed home IP, and
  survive dynamic-IP changes.**
- **Workers are cattle.** Add/remove home boxes freely; Swarm reschedules. Lose
  a home node → the manager and other nodes carry on.
- **State is recoverable** — Dokploy's native backup (below), not bespoke.
- **Control plane is reproducible** — this Terraform fork; capacity retries are
  a loop, not manual clicking.

## Layer 1 — Control plane (this repo)

OCI Always-Free **A1.Flex** VM running Dokploy, provisioned by this fork.

- **Run it locally with tofu** (not the web wizard) so the inevitable
  `Out of host capacity` is a retry loop and the config is version-controlled:
  ```bash
  cd ~/Projects/dokploy-oci-free
  cp terraform.tfvars.example terraform.tfvars   # values below
  # one-time: OCI API creds in ~/.oci/config (generate an API key in the console)
  until tofu apply -auto-approve; do echo "capacity retry…"; sleep 120; done
  ```
- **Variables** (`terraform.tfvars`):
  | var | value |
  | --- | --- |
  | `ssh_authorized_keys` | contents of `~/.ssh/dokploy-oci.pub` |
  | `compartment_id` | tenancy OCID (Profile → Tenancy → OCID) |
  | `source_image_id` | Ubuntu 22.04 image OCID for the region |
  | `availability_domain_main` | e.g. `xxxx:US-SANJOSE-1-AD-1` |
  | `num_worker_instances` | **0** — workers come from home, not OCI |
  | `ocpus` / `memory_in_gbs` | `2` / `12` (whole free budget on the manager) |
  | `use_reserved_public_ip` | `true` — stable IP across instance recreation |
- After apply: SSH in with `~/.ssh/dokploy-oci`, `bin/dokploy-main.sh` installs
  Dokploy. Panel at `https://<public-ip>:3000`.

> Region: home region is fixed at signup (currently San Jose). The panel's
> region is not latency-critical — apps run on the home workers. Only make a new
> AU-home-region account if an AU *control plane* is genuinely required.

## Layer 2 — Tailscale mesh

Put the control plane and every worker on one tailnet.

- On the OCI VM: `curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up`
- Tag them (e.g. `tag:dokploy`) so ACLs are simple.
- Swarm will advertise over the tailnet IPs (Layer 3).

## Layer 3 — Home Swarm workers (bare metal / Docker / Colima)

Each home box becomes a worker. Works on:
- **Bare-metal Linux** — native Docker.
- **macOS** — Docker Desktop or **Colima** (`colima start`) provides the Docker
  engine; the VM joins the Swarm.

Per worker:
```bash
# 1. Docker (native, Docker Desktop, or `colima start`)
# 2. Tailscale
curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up
# 3. join the Swarm over the tailnet, advertising the tailnet IP
docker swarm join --token <worker-token> \
  --advertise-addr <this-node-tailnet-ip> \
  <manager-tailnet-ip>:2377
```
Get `<worker-token>` from the manager: `docker swarm join-token worker`.
The manager must have been `docker swarm init --advertise-addr <manager-tailnet-ip>`
(Dokploy sets up Swarm; re-init advertise-addr to the tailnet IP if needed).

Dokploy then sees the nodes and can schedule services across them. Label nodes
(`docker node update --label-add`) to pin workloads to specific hardware.

## Layer 4 — Backups (Dokploy native → Cloudflare R2, free)

Full-instance backup (Postgres config/state + `/etc/dokploy`), so the control
plane is disposable:

1. Create a Cloudflare **R2** bucket + API token (10 GB free).
2. Dokploy → add it as an S3 **Destination**.
3. Web Server → Backups → schedule daily (cron).
4. DR: fresh Layer-1 apply → Dokploy → Restore from R2.

## Layer 5 — CI/CD

Because the control plane is public, either works:
- **Native:** Dokploy GitHub App → push auto-deploys (webhook reaches the public
  panel — no self-hosted runner needed).
- **Explicit:** the `deploy-dokploy` reusable workflow in `drksci/.github`
  (`gh workflow run deploy-dokploy.yml` or on push).

## Reprovision / DR runbook

1. `tofu apply` (retry loop) → new OCI control plane.
2. `bin/dokploy-main.sh` → Dokploy up.
3. Dokploy → Restore latest R2 backup → all projects/apps/domains/env return.
4. Re-join home workers (Layer 3) — tokens rotate, so re-run `swarm join`.
5. Tailscale + reserved public IP mean DNS/ingress need no changes.
