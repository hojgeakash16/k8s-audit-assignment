# K8S Audit

Fill this in after you run `./audit.sh`. The script is only a hint. Your evidence below is what gets scored.

## My app

- Name: Sample Voting App

- Repo link: https://github.com/hojgeakash16/k8s-audit-assignment.git

- Tiers (frontend / API / database or cache, and what each one is built with):
  - Vote: Python Flask application served by Gunicorn.
  - Result: Node.js / Express application.
  - Worker: .NET background worker.
  - Redis: Redis cache/queue.
  - PostgreSQL: PostgreSQL 15 database.

- Kubernetes manifests are in (folder): 'sample-app/kubernetes/'

- How to run it from a fresh machine (every command, in order, starting from `kind create cluster`):
  kind create cluster --name audit --config kind/kind-config.yaml

kubectl get nodes

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.2/deploy/static/provider/kind/deploy.yaml

cd sample-app/vote
docker build -t vote:1.0 .

cd ../result
docker build -t result:1.0 .

cd ../worker
docker build -t sample-worker:1.0 .

kind load docker-image vote:1.0 --name audit
kind load docker-image result:1.0 --name audit
kind load docker-image sample-worker:1.0 --name audit

cd ../kubernetes
kubectl apply -f .

kubectl get pods -n sample-app-ns
kubectl get services -n sample-app-ns
kubectl get ingress -n sample-app-ns

- How to open it (URL, or port-forward command):
  curl -I -H "Host: vote.local" http://localhost/
  curl -I -H "Host: result.local" http://localhost/

## Before you submit

- [ ] "My app" is filled in and the run steps work on a fresh cluster
- [ ] Every row has a status; every ✅ has evidence and a one-line reason
- [ ] I ran `./audit.sh` and I can explain every ❌ and ⚠️
- [ ] Manifests are in the repo and the links in the table work

## How to fill the table

- **Status**: ✅ used and working. ⚠️ tried, or only partly used (say what is missing). ❌ not used.
- **Evidence**: a link to the file in your repo (with line numbers if the file is long), or pasted command output in a code block. Not a screenshot.
- **Why I used it in my app**: one line. What would break or get worse without it?
- A tick without evidence does not count. A concept that does nothing for your app does not count either.
- Leave the last column as it is. It is there to help you.

## Audit

| Concept | Status (✅ / ⚠️ / ❌) | Evidence | Why I used it in my app | Where to look |
|---|---|---|---|---|
| Deployment + ReplicaSet | ✅ | kubectl get deployments -n sample-app-ns shows 5 Deployments. Each Deployment manages a ReplicaSet and maintains the desired number of Pods | Deployments keep the application components running and provide replica management and rolling updates. | sample-app/kubernetes/ |
| Service | ✅ | kubectl get svc -n sample-app-ns shows 4 Services: vote, result, redis, and db | Services provide stable networking and DNS names between application components. | sample-app/kubernetes/ |
| Namespace | ✅ | kubectl get namespaces shows sample-app-ns | Keeps the application's Kubernetes resources isolated from other namespaces. | sample-app/kubernetes/ |
| Labels and selectors | ✅ | kubectl get endpoints -n sample-app-ns` shows endpoints for all 4 application Services, confirming that Service selectors match Pod labels. | Labels and selectors connect Services to the correct application Pods. | Deployment and Service manifests in sample-app/kubernetes/ | sample-app/kubernetes/ |
| Rolling update + rollback | ✅ | vote-deployment was updated from 'vote:1.0` to `vote:2.0`, the rollout completed successfully, and `kubectl rollout undo` restored `vote:1.0` | Allows application versions to be updated without recreating the entire Deployment and provides rollback capability. | sample-app/kubernetes/vote-deployment.yml |
| ConfigMap | ✅ | `vote-config` contains `OPTION_A=Cats` and `OPTION_B=Dogs` and is consumed by the vote Deployment. | Keeps non-sensitive application configuration outside the container image. | `sample-app/kubernetes/` |
| Secret | ✅ | `postgres-secret` is an Opaque Secret containing the PostgreSQL password. Application Pods consume it using `secretKeyRef` | Keeps sensitive database credentials separate from Deployment configuration. | sample-app/kubernetes/ |
| Requests and limits | ✅ | Audit output: `7 of 7 containers have cpu+memory requests and limits`. | Prevents uncontrolled resource consumption and allows Kubernetes to make better scheduling decisions. | Deployment and CronJob manifests in `sample-app/kubernetes/` |
| Probes (liveness + readiness) | ✅ | Audit output: `6 of 7 containers have both probes`. The worker does not expose an HTTP endpoint, so HTTP probes are not applicable to it. | Readiness controls when traffic is sent to an application, while liveness allows Kubernetes to restart unhealthy containers. | Vote, result, Redis and PostgreSQL Deployment manifests |
| PVC | ✅ | Audit output: `1 bound PVC(s)`. PostgreSQL uses `sample-app-pvc` mounted at `/var/lib/postgresql/data`. | Provides persistent storage for PostgreSQL data. | PostgreSQL Deployment and PVC manifest |
| Ingress | ✅ | `sample-app-ingress` routes `vote.local` to `vote` and `result.local` to `result`. Both routes returned HTTP 200 using Host headers. | Provides HTTP routing to the application Services through the nginx Ingress controller. | Ingress manifest in `sample-app/kubernetes/` |
| Multi-node kind cluster | ✅ | Audit output: `3 node(s)`. The cluster contains `audit-control-plane`, `audit-worker`, and `audit-worker2`. | Demonstrates Kubernetes workloads running on a multi-node cluster. | `kind/kind-config.yaml` |
| HPA (stretch) | ✅ | Audit output: `1 HPA(s)`. `vote-hpa` targets `vote-deployment` with a 50% CPU utilization target, minimum 2 and maximum 5 replicas. | Automatically adjusts vote application replicas based on CPU utilization. | `sample-app/kubernetes/vote-hpa.yml` |
| RBAC + ServiceAccount (stretch) | ❌ | [RBAC examples](https://github.com/LondheShubham153/kubestarter/blob/main/RBAC/) <!-- TODO: add video timestamp --> |
| CronJob (stretch) | ✅ | Audit output: `1 cronjob(s)`. `sample-app-audit` periodically creates Jobs that complete successfully. Example log: `Kubernetes audit CronJob executed`. | Demonstrates scheduled Kubernetes workloads and periodic audit execution. | `sample-app/kubernetes/audit-cronjob.yml` |
| GitHub Actions deploying to kind (stretch) | | | | [helm/kind-action](https://github.com/helm/kind-action), [example workflow](examples/sample-app-k8s/.github/workflows/kind-deploy.yml) <!-- TODO: add video timestamp --> |

## Score

- Must-have concepts with ✅ and evidence (12 max):
- Stretch concepts with ✅ and evidence (4 max):
- Total (pass at 10 or more):

## What was hard / what I would change

A few lines, in your own words.
