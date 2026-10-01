# K8S Audit (filled example)

This is a finished audit for the sample voting app. The manifests it points to are in [sample-app-k8s/](sample-app-k8s/). Your app will be different, so copy the style, not the answers: every row has proof, and every "why" says what would break without the concept.

## My app

- Name: example voting app (cats vs dogs, here "Kubernetes vs Docker Swarm")
- Repo link: this repo, `sample-app/` (code) and `examples/sample-app-k8s/` (manifests)
- Tiers: `vote` (Python/Flask, web) and `result` (Node.js, web) are the frontends. `worker` (.NET) moves votes from `redis` (queue) to `db` (Postgres).
- Kubernetes manifests are in: `examples/sample-app-k8s/`
- How to run it from a fresh machine:

```bash
kind create cluster --config kind/kind-config.yaml
# install ingress-nginx and metrics-server with the commands in kind/README.md
cd sample-app
docker build -t vote:1.0 ./vote && docker build -t result:1.0 ./result && docker build -t worker:1.0 ./worker
docker tag vote:1.0 vote:2.0
kind load docker-image vote:1.0 vote:2.0 result:1.0 worker:1.0 --name audit
cd .. && kubectl apply -f examples/sample-app-k8s/
```

- How to open it: <http://vote.localtest.me> and <http://result.localtest.me> (add `:8080` if you mapped host port 8080 in the kind config).

## Audit

| Concept | Status | Evidence | Why I used it in my app | Where to look |
|---|---|---|---|---|
| Deployment + ReplicaSet | ✅ | [5 Deployments](sample-app-k8s/03-app-tier.yaml). `kubectl -n vote get rs -l app=vote` shows two ReplicaSets, one per version (output in the rollback row). | `vote` runs 2 replicas, so one pod can die and voting keeps working. | [nginx Deployment](https://github.com/LondheShubham153/kubestarter/blob/main/examples/nginx/deployment.yml), [ReplicaSet](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/nginx/replicasets.yml) <!-- TODO: add video timestamp --> |
| Service | ✅ | [4 ClusterIP Services](sample-app-k8s/03-app-tier.yaml#L38-L45): `vote`, `result`, `db`, `redis` | The app code connects to the hostnames `db` and `redis`, so the Services must have exactly those names. `worker` has no Service because nothing calls it. | [nginx Service](https://github.com/LondheShubham153/kubestarter/blob/main/examples/nginx/service.yml) <!-- TODO: add video timestamp --> |
| Namespace | ✅ | [00-namespace.yaml](sample-app-k8s/00-namespace.yaml) | Everything lives in `vote`. One `kubectl delete ns vote` cleans up the whole app between tests. | [namespace manifest](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/nginx/namespace.yml), [k8s docs](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) <!-- TODO: add video timestamp --> |
| Labels and selectors | ✅ | Every Service has endpoints, which only happens when its selector matches pod labels. Output below. | Pods carry `app` and `tier` labels. Services and Deployments select on `app`. `tier` lets me run `kubectl get pods -l tier=data`. | [commands: namespaces, labels, selectors](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/README.md), [k8s docs](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) <!-- TODO: add video timestamp --> |
| Rolling update + rollback | ✅ | [strategy](sample-app-k8s/03-app-tier.yaml#L9-L11) and output below | I moved `vote` from `vote:1.0` to `vote:2.0` with `maxUnavailable: 0`, so there are always 2 ready pods. Then I rolled back with `rollout undo`. | [rolling update](https://github.com/LondheShubham153/kubestarter/blob/main/Deployment_Strategies/Rolling-Update-Deployment/) (rollback is not covered there, see [k8s docs](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)) <!-- TODO: add video timestamp --> |
| ConfigMap | ✅ | [vote-config](sample-app-k8s/01-config.yaml#L1-L10), used by `vote` with [envFrom](sample-app-k8s/03-app-tier.yaml#L25-L26) | The two voting options are config, not code. I can change the question without rebuilding the image. | [MySQL ConfigMap](https://github.com/LondheShubham153/kubestarter/blob/main/examples/mysql/configMap.yml) <!-- TODO: add video timestamp --> |
| Secret | ✅ | [db-credentials](sample-app-k8s/01-config.yaml#L11-L18), read through `secretKeyRef` by [db](sample-app-k8s/02-data-tier.yaml#L34-L37), [result](sample-app-k8s/03-app-tier.yaml#L65-L69), [worker](sample-app-k8s/03-app-tier.yaml#L108-L112) and the [backup job](sample-app-k8s/07-backup-cronjob.yaml#L29-L30) | One password, defined once, used by 4 workloads. A Secret is not encryption (it is only base64), but it keeps the password out of the image and out of the Deployments. | [MySQL Secret](https://github.com/LondheShubham153/kubestarter/blob/main/examples/mysql/secrets.yml) <!-- TODO: add video timestamp --> |
| Requests and limits | ✅ | Every container has all four values, for example [db](sample-app-k8s/02-data-tier.yaml#L47-L50). `./audit.sh vote` says `6 of 6 containers`. | Without requests the scheduler cannot spread pods well. Without limits one busy `vote` pod could starve Postgres on the same node. The HPA also needs a CPU request to calculate a percentage. | [Deployment with resources](https://github.com/LondheShubham153/kubestarter/blob/main/HPA_VPA/apache-deployment.yml), [k8s docs](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) <!-- TODO: add video timestamp --> |
| Probes (liveness + readiness) | ✅ | `db`, `redis`, `vote`, `result` have both, for example [vote](sample-app-k8s/03-app-tier.yaml#L27-L33). `worker` has none on purpose: it has no port, so there is nothing to call. | Readiness keeps a starting `vote` pod out of the Service until Flask answers. Rolling update depends on it. Liveness restarts a hung pod. | [k8s docs](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/), [liveness example](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/django-notes-app/k8s/deployment.yml) <!-- TODO: add video timestamp --> |
| PVC | ✅ | [db-data](sample-app-k8s/02-data-tier.yaml#L1-L12) and output below | Votes must survive a Postgres pod restart. I deleted the pod and the count was the same afterwards. | [PVC](https://github.com/LondheShubham153/kubestarter/blob/main/PersistentVolumes/PersistentVolumeClaim.yaml), [MySQL volumes](https://github.com/LondheShubham153/kubestarter/blob/main/examples/mysql/persistentVols.yml) (kind creates the volume for you, see [kind/README.md](../kind/README.md)) <!-- TODO: add video timestamp --> |
| Ingress | ✅ | [04-ingress.yaml](sample-app-k8s/04-ingress.yaml). `curl http://vote.localtest.me` returns 200. | One entry point on port 80 for two web apps, split by hostname. No NodePort per app. | [Ingress examples](https://github.com/LondheShubham153/kubestarter/blob/main/Ingress/) (written for minikube, use [kind/README.md](../kind/README.md) for the controller), [Ingress manifest](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/nginx/ingress.yml) <!-- TODO: add video timestamp --> |
| Multi-node kind cluster | ✅ | [kind-config.yaml](../kind/kind-config.yaml). `kubectl get nodes` shows 3 nodes. The `vote` pods ran on two different workers. | I wanted to see pods spread across nodes, and `kind load docker-image` has to reach all of them. | [kubestarter kind config](https://github.com/LondheShubham153/kubestarter/blob/main/kind-cluster/kind-config.yml), [our config](../kind/kind-config.yaml) <!-- TODO: add video timestamp --> |
| HPA (stretch) | ✅ | [05-hpa.yaml](sample-app-k8s/05-hpa.yaml). `kubectl -n vote get hpa` shows `cpu: 6%/60%`, so metrics-server works. | `vote` is the part that gets traffic. 2 to 5 replicas at 60% CPU. | [HPA manifest](https://github.com/LondheShubham153/kubestarter/blob/main/HPA_VPA/apache-hpa.yml), [metrics-server steps](https://github.com/LondheShubham153/kubestarter/blob/main/HPA_VPA/README.md) <!-- TODO: add video timestamp --> |
| RBAC + ServiceAccount (stretch) | ✅ | [06-rbac.yaml](sample-app-k8s/06-rbac.yaml) and output below | `vote-app` is the identity of the vote pods, with no API token mounted because the app never calls the API. `viewer` can read pods and logs, nothing else. | [RBAC examples](https://github.com/LondheShubham153/kubestarter/blob/main/RBAC/) <!-- TODO: add video timestamp --> |
| CronJob (stretch) | ✅ | [07-backup-cronjob.yaml](sample-app-k8s/07-backup-cronjob.yaml) and output below | A nightly `pg_dump` into its own PVC. Votes are the only data the app has. | [CronJob manifest](https://github.com/LondheShubham153/kubernetes-in-one-shot/blob/master/nginx/cron-job.yml), [k8s docs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) <!-- TODO: add video timestamp --> |
| GitHub Actions deploying to kind (stretch) | ⚠️ | [kind-deploy.yml](sample-app-k8s/.github/workflows/kind-deploy.yml). Written, and `REPO=examples/sample-app-k8s ./audit.sh` detects it. I ran its build, load, apply and smoke-test steps by hand, but never ran the workflow on GitHub. | I want every pull request to prove the manifests work on a clean 3-node cluster. | [helm/kind-action](https://github.com/helm/kind-action), [example workflow](sample-app-k8s/.github/workflows/kind-deploy.yml) <!-- TODO: add video timestamp --> |

Score: 15 ✅ and 1 ⚠️. Must-have 12/12, stretch 3/4. Total 15.

## Evidence output

Labels and selectors (each Service has endpoints):

```
$ kubectl -n vote get endpointslices
NAME           ADDRESSTYPE   PORTS   ENDPOINTS                             AGE
db-cr6wb       IPv4          5432    10.244.2.34                           32s
redis-6f7sg    IPv4          6379    10.244.1.43                           32s
result-2dlb7   IPv4          80      10.244.1.44                           32s
vote-bc2nn     IPv4          80      10.244.1.46,10.244.2.36,10.244.1.47   32s
```

Rolling update and rollback:

```
$ kubectl -n vote set image deploy/vote vote=vote:2.0
deployment.apps/vote image updated
$ kubectl -n vote rollout history deploy/vote
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
$ kubectl -n vote rollout undo deploy/vote
deployment.apps/vote rolled back
$ kubectl -n vote get rs -l app=vote
NAME              DESIRED   CURRENT   READY   AGE
vote-74d9d54556   0         0         0       20s     <- vote:2.0, scaled to 0 after undo
vote-77cb8b46d7   2         2         2       31s     <- vote:1.0, back in service
```

PVC keeps the votes (3 votes before and after deleting the Postgres pod):

```
$ kubectl -n vote get pvc db-data
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
db-data   Bound    pvc-c0675d7d-db6a-49e2-a88b-9f01423d82da   1Gi        RWO            standard
$ kubectl -n vote exec deploy/db -- psql -U postgres -tc "select vote, count(*) from votes group by 1"
 a    |     3
$ kubectl -n vote delete pod -l app=db
pod "db-5c997978b7-hj8bd" deleted from vote namespace
$ kubectl -n vote exec deploy/db -- psql -U postgres -tc "select vote, count(*) from votes group by 1"
 a    |     3
```

(The `db-backups` PVC shows `Pending` until the first backup Job runs. That is normal for the `standard` StorageClass: it waits for a pod to use the volume.)

RBAC:

```
$ kubectl auth can-i list pods -n vote --as=system:serviceaccount:vote:viewer
yes
$ kubectl auth can-i delete pods -n vote --as=system:serviceaccount:vote:viewer
no
```

CronJob (run once by hand instead of waiting for 2am):

```
$ kubectl -n vote create job --from=cronjob/db-backup db-backup-manual
job.batch/db-backup-manual created
$ kubectl -n vote logs job/db-backup-manual
-rw-r--r--    1 root     root          1232 Oct  1 13:31 votes-20261001-133111.sql
```

The audit script, run on the finished app with the workflow folder passed in:

```
$ REPO=examples/sample-app-k8s ./audit.sh vote
```

See [TESTING.md](../TESTING.md) for the full output.

## What was hard / what I would change

- The kind ingress-nginx manifest did not pin the controller to the node with ports 80/443. On 3 nodes it landed on a worker and every request was reset. I pinned it with a `nodeSelector` (see `kind/README.md`).
- `worker` has no probes. A better version would add a small health file or endpoint and probe that.
- Postgres is a Deployment with `strategy: Recreate` and one replica. For more than one replica this would need a StatefulSet.
