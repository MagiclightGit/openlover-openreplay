# OpenReplay Ubuntu 24.04 Single-Node Deploy

This is a tested single-machine deployment flow for this OpenReplay fork.

Domain used below:

```bash
monitor.playshot.ai
```

Email used below:

```bash
libiaoaire@163.com
```

## 1. Install basic packages

Run line by line:

```bash
sudo apt update
sudo apt install -y git wget curl certbot
```

## 2. Install OpenReplay CLI

```bash
sudo wget https://raw.githubusercontent.com/openreplay/openreplay/main/scripts/helmcharts/openreplay-cli -O /bin/openreplay
sudo chmod +x /bin/openreplay
```

## 3. Install OpenReplay

```bash
openreplay -i monitor.playshot.ai
```

Wait for the install to finish.

## 4. Apply TLS certificate

Do not use `bash certmanager.sh` for this environment.

Use `certbot` and then import the certificate into Kubernetes.

```bash
export DOMAIN="monitor.playshot.ai"
export EMAIL="libiaoaire@163.com"
```

Remove failed cert-manager state if it exists:

```bash
kubectl -n app annotate ingress frontend-openreplay-master cert-manager.io/cluster-issuer- --overwrite
kubectl -n app delete certificate openreplay-ssl --ignore-not-found
kubectl -n app delete certificaterequest --all --ignore-not-found
kubectl -n app delete order --all --ignore-not-found
kubectl -n app delete challenge --all --ignore-not-found
kubectl -n app delete secret openreplay-ssl --ignore-not-found
```

Stop k3s and fully release ports `80/443`:

```bash
sudo systemctl stop k3s
sudo /usr/local/bin/k3s-killall.sh
```

Request the certificate:

```bash
sudo certbot certonly --standalone --preferred-challenges http -d "$DOMAIN" -m "$EMAIL" --agree-tos --no-eff-email -v
```

If `certbot` says an existing certificate already exists, choose:

```text
2
```

to renew and replace it.

## 5. Start k3s again and import TLS secret

```bash
sudo systemctl start k3s
kubectl wait --for=condition=Ready nodes --all --timeout=180s
kubectl -n app create secret tls openreplay-ssl --cert="/etc/letsencrypt/live/$DOMAIN/fullchain.pem" --key="/etc/letsencrypt/live/$DOMAIN/privkey.pem" --dry-run=client -o yaml | kubectl apply -f -
kubectl -n app rollout restart deployment openreplay-ingress-nginx-controller
kubectl -n app rollout status deployment openreplay-ingress-nginx-controller --timeout=180s
```

## 6. Fix ingress issues required by this fork

Remove the problematic annotation on the frontend master ingress:

```bash
kubectl -n app annotate ingress frontend-openreplay-master nginx.org/proxy-set-headers- --overwrite
```

Fix the `images-openreplay` ingress so it does not take over the whole host:

```bash
kubectl -n app annotate ingress images-openreplay nginx.org/mergeable-ingress-type=minion --overwrite
```

Reload the ingress controller:

```bash
kubectl -n app rollout restart deployment openreplay-ingress-nginx-controller
kubectl -n app rollout status deployment openreplay-ingress-nginx-controller --timeout=180s
```

## 7. Verify deployment

```bash
kubectl -n app get ingress
kubectl -n app get secret openreplay-ssl
curl -I --resolve monitor.playshot.ai:80:101.47.182.200 http://monitor.playshot.ai/
curl -Ik --resolve monitor.playshot.ai:443:101.47.182.200 https://monitor.playshot.ai/
```

If everything is correct, you should be able to open:

```bash
https://monitor.playshot.ai
```

## 8. Reapply after restart or redeploy

If you reinstall or rerun `openreplay -i`, re-run these fixes:

```bash
kubectl -n app annotate ingress frontend-openreplay-master nginx.org/proxy-set-headers- --overwrite
kubectl -n app annotate ingress images-openreplay nginx.org/mergeable-ingress-type=minion --overwrite
kubectl -n app rollout restart deployment openreplay-ingress-nginx-controller
kubectl -n app rollout status deployment openreplay-ingress-nginx-controller --timeout=180s
```

## 9. Notes

- `certmanager.sh` was not reliable in this environment because the ACME HTTP-01 challenge returned `404`.
- The final working path was: `certbot --standalone` -> create Kubernetes TLS secret -> fix ingress annotations.
- If HTTPS works but the site returns `404`, check whether `images-openreplay` has taken the same host.
