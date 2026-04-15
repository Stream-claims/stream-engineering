---
name: tailscale-vpn
description: Tailscale CLI commands and Stream VPN troubleshooting. Use when switching environments (dev/staging/prod), debugging 403 errors on Stream apps, or managing Tailscale exit nodes.
---

# Tailscale CLI & Stream VPN

## Quick Reference

| Environment | Exit Node | Access |
|-------------|-----------|--------|
| **Dev** | `dev` exit node | Dev environments only |
| **Staging** | `staging` exit node | Staging environments only |
| **Production** | `production` exit node OR off VPN entirely | If outside US: use `production` node. |

**Environments are isolated** — you cannot browse from one stage to another. A `403 Forbidden` on a Stream environment almost always means you're on the wrong VPN node or are off VPN altogether.

## Common Commands

```bash
# Check current status
tailscale status

# List available exit nodes
tailscale exit-node list

# Switch to dev
tailscale set --exit-node=<dev-node>

# Switch to staging
tailscale set --exit-node=<staging-node>

# Disconnect VPN (for US-based prod access)
tailscale set --exit-node=

# Connect/disconnect entirely
tailscale up
tailscale down
```

## Troubleshooting

**Getting 403 on a Stream environment?**
1. Run `tailscale status` to check your current exit node
2. Match the exit node to the environment you're trying to reach (see table above)
3. Switch exit nodes if mismatched
