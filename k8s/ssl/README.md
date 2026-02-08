# SSL Certificate for Ingress (OpenSSL)

Create a TLS certificate and key with OpenSSL, then create a Kubernetes TLS secret for the Ingress in the **frontend** namespace.

## 1. Generate certificate and key (self-signed)

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=buzzboard.local"
```

This creates:
- `tls.key` — private key
- `tls.crt` — certificate

Use the same CN (or SAN) as the host in your Ingress (e.g. `buzzboard.local`).

## 2. Create TLS secret in frontend and backend namespaces

The frontend Ingress (path `/`) and the backend Ingress (paths `/api/reactions`, `/api/mood`) both use the same host and TLS. Create the secret in **both** namespaces:

```bash
kubectl create secret tls ingress-tls --cert=tls.crt --key=tls.key -n frontend
kubectl create secret tls ingress-tls --cert=tls.crt --key=tls.key -n backend
```

## 3. (Optional) Add to /etc/hosts

For local testing with `buzzboard.local`:

```
127.0.0.1 buzzboard.local
```

Then open https://buzzboard.local in the browser (accept the self-signed cert warning).

## Note

Do not commit `tls.key` or `tls.crt` to the repo. The Ingress manifests reference the secret name `ingress-tls`; create the secret in both `frontend` and `backend` namespaces after deploying (or before first access). See the main [README.md](../../README.md) "Deploy to Kubernetes" section for the full apply order.
