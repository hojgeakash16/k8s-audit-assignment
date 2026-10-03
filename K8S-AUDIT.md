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
| Deployment + ReplicaSet | ✅ | kubectl get deployments -n sample-app-ns` shows 5 Deployments. Each Deployment manages a ReplicaSet and maintains the desired number of Pods | Deployments keep the application components running and provide replica management and rolling updates. |  | `sample-app/kubernetes/` |<!-- TODO: add video timestamp --> |
| Service | | | | [nginx Service](https://github.com/LondheShubham153/kubestarter/blob/main/examples/nginx/service.yml) <!-- TODO: add video timestamp --> |
| Namespace | | | | [namespace manifest](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/nginx/namespace.yml), [k8s docs](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) <!-- TODO: add video timestamp --> |
| Labels and selectors | | | | [commands: namespaces, labels, selectors](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/README.md), [k8s docs](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) <!-- TODO: add video timestamp --> |
| Rolling update + rollback | | | | [rolling update](https://github.com/LondheShubham153/kubestarter/blob/main/Deployment_Strategies/Rolling-Update-Deployment/) (rollback is not covered there, see [k8s docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)) <!-- TODO: add video timestamp --> |
| ConfigMap | | | | [MySQL ConfigMap](https://github.com/LondheShubham153/kubestarter/blob/main/examples/mysql/configMap.yml) <!-- TODO: add video timestamp --> |
| Secret | | | | [MySQL Secret](https://github.com/LondheShubham153/kubestarter/blob/main/examples/mysql/secrets.yml) <!-- TODO: add video timestamp --> |
| Requests and limits | | | | [Deployment with resources](https://github.com/LondheShubham153/kubestarter/blob/main/HPA_VPA/apache-deployment.yml), [k8s docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) <!-- TODO: add video timestamp --> |
| Probes (liveness + readiness) | | | | [k8s docs](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/), [liveness example](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/django-notes-app/k8s/deployment.yml) <!-- TODO: add video timestamp --> |
| PVC | | | | [PVC](https://github.com/LondheShubham153/kubestarter/blob/main/PersistentVolumes/PersistentVolumeClaim.yaml), [MySQL volumes](https://github.com/LondheShubham153/kubestarter/blob/main/examples/mysql/persistentVols.yml) (kind creates the volume for you, see [kind/README.md](kind/README.md)) <!-- TODO: add video timestamp --> |
| Ingress | | | | [Ingress examples](https://github.com/LondheShubham153/kubestarter/blob/main/Ingress/) (written for minikube, use [kind/README.md](kind/README.md) for the controller), [Ingress manifest](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/nginx/ingress.yml) <!-- TODO: add video timestamp --> |
| Multi-node kind cluster | | | | [kubestarter kind config](https://github.com/LondheShubham153/kubestarter/blob/main/kind-cluster/kind-config.yml), [our config](kind/kind-config.yaml) <!-- TODO: add video timestamp --> |
| HPA (stretch) | | | | [HPA manifest](https://github.com/LondheShubham153/kubestarter/blob/main/HPA_VPA/apache-hpa.yml), [metrics-server steps](https://github.com/LondheShubham153/kubestarter/blob/main/HPA_VPA/README.md) <!-- TODO: add video timestamp --> |
| RBAC + ServiceAccount (stretch) | | | | [RBAC examples](https://github.com/LondheShubham153/kubestarter/blob/main/RBAC/) <!-- TODO: add video timestamp --> |
| CronJob (stretch) | | | | [CronJob manifest](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/nginx/cron-job.yml), [k8s docs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) <!-- TODO: add video timestamp --> |
| GitHub Actions deploying to kind (stretch) | | | | [helm/kind-action](https://github.com/helm/kind-action), [example workflow](examples/sample-app-k8s/.github/workflows/kind-deploy.yml) <!-- TODO: add video timestamp --> |

## Score

- Must-have concepts with ✅ and evidence (12 max):
- Stretch concepts with ✅ and evidence (4 max):
- Total (pass at 10 or more):

## What was hard / what I would change

A few lines, in your own words.
