# infra

Personal server infrastructure managed with k3s + ArgoCD.

## Stack

- **k3s** — single-node Kubernetes
- **Traefik** — reverse proxy + automatic TLS via cert-manager
- **ArgoCD** — GitOps, push to main to deploy
- **Sealed Secrets** — encrypted secrets safe to commit
- **Gatus** — status page at [status.flipez.de](https://status.flipez.de)

## Services

| Service | URL |
|---|---|
| SpeziDB | [spezidb.de](https://spezidb.de) |
| Matrix | [matrix.auch.cool](https://matrix.auch.cool) |
| TheLounge | [irc.auch.cool](https://irc.auch.cool) |
| Rallly | [rallly.auch.cool](https://rallly.auch.cool) |
| Trek | [trek.auch.cool](https://trek.auch.cool) |

## Adding a service

See [CLAUDE.md](CLAUDE.md).
