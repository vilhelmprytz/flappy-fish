# Deploying flappy-fish

Push to `main` → CI builds an arm64 image, installs cert-manager if the cluster
lacks it, applies the ClusterIssuers, and deploys.

Assumes a Pi running k3s with Tailscale up. One-time setup:

## 1. Let k3s answer on its tailnet name

Its serving cert covers localhost and the LAN IP only.

```bash
echo "tls-san: [$(tailscale status --json | jq -r '.Self.DNSName' | sed 's/\.$//')]" \
  | sudo tee -a /etc/rancher/k3s/config.yaml
sudo rm -f /var/lib/rancher/k3s/server/tls/serving-kube-apiserver.{crt,key}
sudo systemctl restart k3s
```

## 2. Forward ports

TCP 80 and 443 → the Pi. Not 6443; CI reaches the API over the tailnet.


## 3. Secrets

All three as secrets, not variables — this repo is public and workflow logs are
world-readable.

```bash
gh secret set TS_AUTHKEY
gh secret set ACME_EMAIL

TS=$(ssh k3s-home "sudo tailscale status --json | jq -r '.Self.DNSName'" | sed 's/\.$//')
ssh k3s-home 'sudo cat /etc/rancher/k3s/k3s.yaml' \
  | sed "s|127.0.0.1|${TS}|" | base64 | tr -d '\n' | gh secret set KUBECONFIG_B64
```
## 4. Push

Push and it should work!
