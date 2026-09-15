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
## 4. Split-horizon DNS

If you have split-horizon DNS setup, you might need to do this to allow in-cluster look-ups.

```bash
kubectl apply -f - <<YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  k8s-signed-zone.override: |
    template IN A k8s.signed.zone {
      match "^(.*\\.)?k8s\\.signed\\.zone\\.$"
      answer "{{ .Name }} 60 IN A $(kubectl -n kube-system get svc traefik -o jsonpath='{.spec.clusterIP}')"
      fallthrough
    }
YAML
```

k3s imports `*.override` into its Corefile and the `reload` plugin picks it up.

## 5. Push

Push and it should work!
