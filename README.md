# flappy-fish

Flappy Bird, but a fish. SvelteKit + canvas, served from a Raspberry Pi at
[flappy-fish.k8s.signed.zone](https://flappy-fish.k8s.signed.zone).

```bash
npm install
npm run dev
```

## Pipeline

```mermaid
flowchart LR
  dev([push / PR]) --> ci[GitHub Actions]

  subgraph pi [Raspberry Pi · tailnet]
    k3s[k3s] --> traefik[Traefik + cert-manager] --> app[flappy-fish]
  end

  ci -- docker build --> ghcr[(GHCR)]
  ci -- helm upgrade --> k3s
  ci -- dnscontrol --> dns[DNS provider]
  ghcr -. pull .-> k3s
  dns -.-> traefik
```

Every PR gets its own namespace and hostname,
`flappy-fish-pr-<n>.k8s.signed.zone`, removed when the PR closes.

Cluster setup: [deploy/README.md](deploy/README.md) · chart: [flappy-fish/](flappy-fish/) · DNS: [dnscontrol/](dnscontrol/)
