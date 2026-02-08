# Kubernetes deployment — Ashour Chat Tutorial

This folder contains manifests for the three namespaces (frontend, backend, data). The app is the same as Docker Compose: **Ashour Chat Tutorial** with **Sign in / Sign up**; Reaction Wall and Mood require sign-in. APIs are exposed under the same host via Ingress (`/api/reactions`, `/api/mood`). Before applying you need:

1. **NGINX Ingress Controller** installed in the cluster.
2. **Hosts entry** so `buzzboard.local` (used in [namespace-1-frontend/ingress.yaml](namespace-1-frontend/ingress.yaml)) resolves to the Ingress.

---

## 1. Install NGINX Ingress Controller

Use **one** of the following, depending on your cluster.

### Option A — Official manifest (any cluster)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/cloud/deploy.yaml
```

Replace `controller-v1.11.1` with a [released tag](https://github.com/kubernetes/ingress-nginx/releases) if you want a different version. Wait until the Ingress controller pod is running:

```bash
kubectl get pods -n ingress-nginx
```

### Option B — Minikube

```bash
minikube addons enable ingress
```

### Option C — kind (Kubernetes in Docker)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### Option D — k3d / k3s

k3d/k3s often ship with Traefik. To use **NGINX** Ingress instead, disable Traefik and install the official manifest (Option A), or use your distribution’s docs for the NGINX Ingress Controller.

---

## 2. Get the Ingress controller address

The Ingress must be reachable at an IP (or hostname) so you can point `buzzboard.local` to it.

### Minikube

```bash
minikube ip
```

Use this IP in the hosts file (e.g. `192.168.49.2 buzzboard.local`).

### Other clusters (LoadBalancer or NodePort)

- **LoadBalancer:** get the external IP of the ingress-nginx service:
  ```bash
  kubectl get svc -n ingress-nginx
  ```
  Use the `EXTERNAL-IP` (or `pending` then retry until an IP appears).

- **NodePort / no LoadBalancer:** use any node IP and the NodePort, or use port-forward for local testing:
  ```bash
  kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 443:443 80:80
  ```
  Then in hosts use `127.0.0.1 buzzboard.local`.

---

## 3. Add `buzzboard.local` to your hosts file

The Ingress in [namespace-1-frontend/ingress.yaml](namespace-1-frontend/ingress.yaml) uses:

```yaml
rules:
  - host: buzzboard.local
tls:
  - hosts:
      - buzzboard.local
```

So your machine must resolve **buzzboard.local** to the Ingress controller IP.

### macOS / Linux

Edit `/etc/hosts` with sudo:

```bash
sudo nano /etc/hosts
```

Add one line (replace `INGRESS_IP` with the IP from step 2, or `127.0.0.1` if using port-forward):

```
INGRESS_IP   buzzboard.local
```

Examples:

- Minikube: `192.168.49.2   buzzboard.local`
- Port-forward: `127.0.0.1   buzzboard.local`
- Cloud LoadBalancer: `34.120.xx.xx   buzzboard.local`

Save and exit.

### Windows

1. Open Notepad **as Administrator**.
2. Open: `C:\Windows\System32\drivers\etc\hosts`
3. Add a line: `INGRESS_IP   buzzboard.local` (same rules as above).
4. Save.

---

## 4. TLS (HTTPS)

The Ingress expects a TLS secret named `ingress-tls` in the **frontend** namespace. Create it with your own cert or the self-signed steps in **[ssl/README.md](ssl/README.md)**:

```bash
# After creating tls.crt and tls.key (see ssl/README.md)
kubectl create secret tls ingress-tls --cert=tls.crt --key=tls.key -n frontend
```

Then open **https://buzzboard.local** in the browser and accept the self-signed certificate warning if needed.

---

## 5. Using the app (by Kubernetes)

After all manifests and Ingress are applied:

1. Open **https://buzzboard.local** (ensure it resolves via your hosts file).
2. You should see **Ashour Chat Tutorial** with **Sign in** and **Sign up** in the menu.
3. **Sign up** to create an account, then **Sign in**.
4. Use **Reaction Wall** and **Mood of the Day**; each post and vote is stored with your user. The frontend calls the backend via the same host (`/api/reactions`, `/api/mood`) so CORS and cookies work without extra config.

---

## Summary

| Step | Action |
|------|--------|
| 1 | Install NGINX Ingress Controller (manifest, minikube addon, or distro method). |
| 2 | Get Ingress IP: `minikube ip`, or `kubectl get svc -n ingress-nginx`, or use `127.0.0.1` with port-forward. |
| 3 | Add `INGRESS_IP   buzzboard.local` to `/etc/hosts` (Mac/Linux) or `C:\...\hosts` (Windows). |
| 4 | Create TLS secret `ingress-tls` in namespaces **frontend** and **backend** (see [ssl/README.md](ssl/README.md)). |
| 5 | Apply all manifests (see main [README.md](../README.md) “Deploy to Kubernetes”); then open **https://buzzboard.local** and use Sign up / Sign in. |

**If the API returns 502/503** (e.g. when signing in or loading reactions): the backend NetworkPolicy allows traffic from the Ingress controller namespace only if it has the label `app.kubernetes.io/name=ingress-nginx`. Add it with: `kubectl label namespace ingress-nginx app.kubernetes.io/name=ingress-nginx` (or adjust [backend-networkpolicy.yaml](namespace-2-backend/backend-networkpolicy.yaml) for your Ingress namespace).
