# Azure DevOps Pipeline Setup

The pipeline (`azure-pipelines.yml`) builds frontend, reactions, and mood images, pushes them to **Docker Hub**, then deploys either to **EC2 (Docker Compose)** or to a **Kubernetes** cluster based on the parameter you choose.

- **Agent pool:** Default  
- **Image registry:** Docker Hub  
- **Parameters:** `deployTarget` (ec2-docker-compose | k8s), optional `imageTag`

---

## 1. Variable group: ITGATE

All pipeline variables are taken from the **variable group** named **`ITGATE`**.

In Azure DevOps: **Pipelines → Library → Variable groups → Create variable group** (or use an existing one). Name it **`ITGATE`**, then add:

| Variable | Description | Secret |
|----------|-------------|--------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username (e.g. `myuser`). Images will be `myuser/ashour-chat-frontend`, etc. | No |
| `EC2_DEPLOY_PATH` | (EC2 only) Path on the EC2 server where to copy files and run compose (e.g. `/home/ubuntu/app`). | No |

The pipeline references this group with `variables: - group: ITGATE`. Ensure the pipeline has access to the group (Pipeline permissions in the group settings).

### Pipeline-computed variables (TAG and effectiveTag)

The pipeline also sets two variables **inside the YAML** (not in the variable group). They control which Docker image tag is built and deployed:

| Variable | Meaning |
|----------|--------|
| **`TAG`** | `$[ coalesce(variables['Build.SourceVersion'], variables['Build.BuildId']) ]` — Uses the **commit SHA** (e.g. `a1b2c3d`) if available; otherwise falls back to the **build ID** (e.g. `12345`). Used as a default/fallback identifier. |
| **`effectiveTag`** | `$[ eq(coalesce('${{ parameters.imageTag }}', ''), '') ? variables['Build.BuildId'] : '${{ parameters.imageTag }}' ]` — This is the tag **actually used** for images and deployment: if you pass a **Docker image tag** when running the pipeline (e.g. `v1.0`), that value is used; otherwise the pipeline uses the **build ID** (e.g. `12345`). So: *parameter provided → use it; otherwise → use Build.BuildId*. |

So when you run the pipeline, images are tagged with `effectiveTag` (and often also `latest`). Deploy steps use `effectiveTag` so EC2 or K8s pull the same build you just pushed.

---

## 2. Service connections

### dockerHubConnection (required for build)

1. **Project Settings → Service connections → New service connection → Docker Registry**  
2. **Docker Hub**  
3. Name: **`dockerHubConnection`** (must match pipeline).  
4. Docker ID and password (or access token).  
5. Save.

### EC2 – SSH (required for EC2 deploy)

1. **New service connection → SSH**.  
2. Name: **`EC2-SSH`**.  
3. Host: your EC2 IP or hostname.  
4. Port: 22.  
5. Username: e.g. `ubuntu`.  
6. Private key: paste the private key (or use a variable).  
7. Save.

### orbstack-k8s (required for K8s deploy)

1. **New service connection → Kubernetes**.  
2. Name: **`orbstack-k8s`**.  
3. Server URL and kubeconfig (or Service account) so the agent can run `kubectl`.  
4. Save.

---

## 3. Environments (optional)

The pipeline uses environments **`ec2`** and **`k8s`** for deployment. If they don’t exist, Azure DevOps can create them on first run, or you can add them under **Pipelines → Environments** and optionally set approval checks.

---

## 4. Running the pipeline

1. **Run pipeline** and choose:
   - **Deployment target:** `ec2-docker-compose` or `k8s`
   - **Docker image tag:** leave empty to use build ID, or set e.g. `v1.0`
2. Build stage: builds and pushes three images to Docker Hub.  
3. Deploy stage: either copies compose files to EC2 and runs `docker compose`, or applies K8s manifests (data → backend → frontend) with the new image tags.

---

## 5. EC2 prerequisites

- Docker and Docker Compose installed.  
- `EC2_DEPLOY_PATH` writable by the SSH user.  
- If using private registry: `docker login` on EC2 (or use the same Docker Hub credentials so `docker compose pull` works).

---

## 6. K8s prerequisites

- Cluster has an Ingress controller (e.g. NGINX) if you use the Ingress manifests.  
- TLS secret `ingress-tls` created in the frontend (and backend) namespace(s) as per `k8s/ssl/README.md`.  
- Namespaces `frontend`, `backend`, `data` will be created by the manifests if not present.
