# Testing

Both tests ran on a real 3-node kind cluster made from `kind/kind-config.yaml` (kind v0.32.0, node image v1.36.1, Docker Desktop on an Apple-silicon Mac), with ingress-nginx and metrics-server installed as in `kind/README.md`.

One change for the test machine: Lima was already using port 443 here, so the test cluster mapped host ports 8088 and 8448 instead of 80 and 443. Nothing else differed. The shipped config keeps 80 and 443.

## Test A: sample app

Kubernetized the voting app the way a strong learner would: [examples/sample-app-k8s/](examples/sample-app-k8s/). Checked by hand that votes go through the ingress to Redis, the worker, Postgres and the result page, that votes survive deleting the Postgres pod, that rollout/undo works, that the `viewer` ServiceAccount can list pods but not delete them, and that the backup CronJob writes a dump. I ran every audit with the namespace argument (`./audit.sh vote`) because Test B's app was in the same cluster.

### A1. Finished app

```
$ ./audit.sh vote
context: kind-audit   scope: vote

✅ Deployment + ReplicaSet              5 deployment(s) in app namespaces
✅ Service                              4 service(s)
✅ Namespace                            1 namespace(s) besides default and system ones
✅ Labels and selectors                 4 service(s) have endpoints, so selectors match pod labels
✅ Rolling update + rollback            1 deployment(s) rolled to a new image (rollback itself is not visible, show it in your evidence)
✅ ConfigMap                            1 non-default configmap(s)
✅ Secret                               1 Opaque secret(s)
✅ Requests and limits                  7 of 7 containers have cpu+memory requests and limits
✅ Probes (liveness+readiness)          6 of 7 containers have both probes
✅ PVC                                  1 bound PVC(s)
✅ Ingress                              1 ingress resource(s), 1 running controller pod(s)
✅ Multi-node cluster                   3 node(s)
✅ HPA (stretch)                        1 HPA(s)
✅ RBAC + ServiceAccount (stretch)      2 role/rolebinding(s), 2 custom serviceaccount(s)
✅ CronJob (stretch)                    1 cronjob(s)
❌ GitHub Actions to kind (stretch)     looks for kind create cluster / helm/kind-action in ./.github/workflows

Seen: 15/16  (must 12/12, stretch 3/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
```

Every line matches what is in the cluster. The only ❌ is the GitHub Actions row: the workflow is in `examples/`, not in a `.github/workflows` folder at the repo root, so by default it is not seen. With `REPO=examples/sample-app-k8s ./audit.sh vote` the same cluster scores 16/16.

### A2. Remove things, rerun

Removed: the liveness probes (readiness stayed), the worker's `resources`, and the Ingress. Lines that changed:

```
-✅ Requests and limits                  7 of 7 containers have cpu+memory requests and limits
-✅ Probes (liveness+readiness)          6 of 7 containers have both probes
+❌ Requests and limits                  5 of 6 containers have cpu+memory requests and limits
+❌ Probes (liveness+readiness)          0 of 6 containers have both probes
-✅ Ingress                              1 ingress resource(s), 1 running controller pod(s)
+❌ Ingress                              0 ingress resource(s), 1 running controller pod(s)
-Seen: 15/16  (must 12/12, stretch 3/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
+Seen: 12/16  (must 9/12, stretch 3/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
```

Probes went to ❌ with readiness still present because the check needs both. Then I scaled the ingress controller to 0 and re-created the Ingress (the Ingress row stays ❌: no running controller), deleted the HPA, the CronJob, the Role and RoleBinding (the two ServiceAccounts stayed, and the RBAC row still went ❌ because it needs both), and pointed the `result` Service at a wrong label. Lines that changed from the previous run:

```
-✅ Labels and selectors                 4 service(s) have endpoints, so selectors match pod labels
+✅ Labels and selectors                 3 service(s) have endpoints, so selectors match pod labels
-❌ Ingress                              0 ingress resource(s), 1 running controller pod(s)
+❌ Ingress                              1 ingress resource(s), 0 running controller pod(s)
-✅ HPA (stretch)                        1 HPA(s)
-✅ RBAC + ServiceAccount (stretch)      2 role/rolebinding(s), 2 custom serviceaccount(s)
-✅ CronJob (stretch)                    1 cronjob(s)
+❌ HPA (stretch)                        0 HPA(s)
+❌ RBAC + ServiceAccount (stretch)      0 role/rolebinding(s), 2 custom serviceaccount(s)
+❌ CronJob (stretch)                    0 cronjob(s)
-Seen: 12/16  (must 9/12, stretch 3/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
+Seen: 9/16  (must 9/12, stretch 0/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
```

The wrong selector is visible in the count (4 to 3 services with endpoints). The row stays ✅ because one correct Service is enough. That is on purpose: it is a hint.

Running it against the `default` namespace (where the app is not) shows almost everything missing. Only the node count survives:

```
$ ./audit.sh default | tail -4
❌ CronJob (stretch)                    0 cronjob(s)
❌ GitHub Actions to kind (stretch)     looks for kind create cluster / helm/kind-action in ./.github/workflows

Seen: 1/16  (must 1/12, stretch 0/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
```

After `kubectl apply -f examples/sample-app-k8s/` and restoring the controller the audit returned to 15/16.

### What changed in audit.sh because of Test A

Nothing in the logic. The only edit was widening the name column. The first thing it prints is the kubectl context, because this machine's context was an EKS cluster before `kind create cluster` switched it. Use that line to check you are auditing the right cluster.

## Test B: devboard-starter

Repo: LondheShubham153/devboard-starter, `main` branch. React frontend (Vite preview, proxies `/api` to `backend:8080`), Go/Gin backend, Postgres. That is three tiers, so I did not need to add one. I worked in my own clone and changed nothing upstream. The ported files are copied to [examples/devboard-on-kind/](examples/devboard-on-kind/).

### B1. EKS assumptions in `k8s/eks` and what I used on kind

I first applied `k8s/eks/` unchanged to kind to see what really breaks.

| EKS file | Assumption | What happened on kind | What I did |
|---|---|---|---|
| `04-postgres-storageclass.yml`, `06-postgres-statefulset.yml` | StorageClass `ebs-gp3`, provisioner `ebs.csi.aws.com` | PVC stuck `Pending`, `postgres-0` stuck `Pending` | Deleted the StorageClass, removed `storageClassName` from the claim template. kind's default `standard` (local-path) provisions the volume. |
| `08-…`, `10-…` deployments | Images `trainwithshubham/devboard-*:817b5cf` on Docker Hub (not ECR), tag rewritten by CI | `ErrImagePull: no match for platform in manifest`. The CI images are amd64 only and my kind nodes are arm64. | Built both images locally, tagged `devboard-*:local`, `kind load docker-image`, `imagePullPolicy: IfNotPresent`. This also works for Intel machines. |
| `12-gateway-api.yml` | Gateway API + Envoy Gateway (GatewayClass, Gateway, HTTPRoute) | `no matches for kind "GatewayClass"` (CRDs and controller not installed) | Replaced with one `Ingress` ([12-ingress.yml](examples/devboard-on-kind/12-ingress.yml)) for ingress-nginx, host `devboard.localtest.me`. |
| `13-envoyproxy.yml` | Envoy Service `type: LoadBalancer` with `aws-load-balancer-*` NLB annotations | `no matches for kind "EnvoyProxy"`. A LoadBalancer would stay `<pending>` on kind anyway. | Dropped the file. kind's `extraPortMappings` (80/443) plus ingress-nginx replace the cloud load balancer. |
| `06-postgres-statefulset.yml` | `PGDATA` subdirectory because EBS adds `lost+found` | Harmless on kind | Kept it, updated the comment. |

Not found in this repo: ALB Ingress annotations, IAM role (IRSA) annotations, ECR image paths, `gp2`. The brief expected some of these, but this project uses Gateway API + NLB, EBS gp3 and Docker Hub instead. Same ideas, different names.

Other things I noticed:

- `docs/kubernetes.md` mentions `k8s/kind-up.sh` and `07-gateway.yaml`. Neither exists in the repo.
- The old top-level `k8s/*.yml` files (hostPath PV, `latest` tags) are not the same as `k8s/eks`. I ported from `k8s/eks`, as asked.
- Neither the backend nor the frontend Deployment has probes, and Postgres has a readiness probe only and no `resources`.

### B2. Ported and deployed

`k8s/kind/` deployed to the 3-node cluster. Pods: backend and frontend Running, `postgres-0` Running with a Bound PVC. Through the ingress: UI returns 200, `GET /api/projects` returns the seed data, `POST /api/tasks` creates a task, and the task is still there after `kubectl delete pod postgres-0`.

### B3. Audit as shipped (after the port, before any gap closing)

```
$ ./audit.sh
context: kind-audit   scope: all non-system namespaces

✅ Deployment + ReplicaSet              2 deployment(s) in app namespaces
✅ Service                              3 service(s)
✅ Namespace                            1 namespace(s) besides default and system ones
✅ Labels and selectors                 3 service(s) have endpoints, so selectors match pod labels
❌ Rolling update + rollback            0 deployment(s) past revision 1 (rollback itself is not visible, show it in your evidence)
✅ ConfigMap                            2 non-default configmap(s)
✅ Secret                               1 Opaque secret(s)
❌ Requests and limits                  2 of 3 containers have cpu+memory requests and limits
❌ Probes (liveness+readiness)          0 of 3 containers have both probes
✅ PVC                                  1 bound PVC(s)
✅ Ingress                              1 ingress resource(s), 1 running controller pod(s)
✅ Multi-node cluster                   3 node(s)
❌ HPA (stretch)                        0 HPA(s)
❌ RBAC + ServiceAccount (stretch)      0 role/rolebinding(s), 0 custom serviceaccount(s)
❌ CronJob (stretch)                    0 cronjob(s)
❌ GitHub Actions to kind (stretch)     looks for kind create cluster / helm/kind-action in ./.github/workflows

Seen: 9/16  (must 9/12, stretch 0/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
```

(This run used the first version of the Rolling update check. The row is ❌ either way.)

9 of 16. The app covers Deployment, Service, Namespace, labels, ConfigMap, Secret, PVC, Ingress and the multi-node cluster. It misses rolling update (never updated), requests/limits (Postgres has none, 2 of 3 containers), probes (none have both), and all four stretch items. Just under the pass mark, which is a fair result for an unmodified project.

### B4. Closing two gaps

Added resources and a liveness probe to Postgres, and liveness plus readiness probes to the backend (`/health`) and frontend (`/`). That is two concepts: requests/limits and probes. Changes in [examples/devboard-on-kind/](examples/devboard-on-kind/) files `06`, `08`, `10`. Lines that changed:

```
-❌ Rolling update + rollback            0 deployment(s) past revision 1 (rollback itself is not visible, show it in your evidence)
+✅ Rolling update + rollback            2 deployment(s) past revision 1 (rollback itself is not visible, show it in your evidence)
-❌ Requests and limits                  2 of 3 containers have cpu+memory requests and limits
-❌ Probes (liveness+readiness)          0 of 3 containers have both probes
+✅ Requests and limits                  3 of 3 containers have cpu+memory requests and limits
+✅ Probes (liveness+readiness)          3 of 3 containers have both probes
-Seen: 9/16  (must 9/12, stretch 0/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
+Seen: 12/16  (must 12/12, stretch 0/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
```

### B5. audit.sh problem found on this project, and the fix

Rolling update turned ✅ above even though I had never updated an image. My probe and limits edits had bumped the Deployment revision to 2, and the first version of the check only looked at revision numbers. Any `kubectl apply` of a changed spec would pass it.

Fix: the check now looks at the ReplicaSets of each Deployment and counts it only if the history holds two or more different images. Rerun on the same cluster, before and after a real rollout:

```
-❌ Rolling update + rollback            0 deployment(s) rolled to a new image (rollback itself is not visible, show it in your evidence)
+✅ Rolling update + rollback            1 deployment(s) rolled to a new image (rollback itself is not visible, show it in your evidence)
-Seen: 11/16  (must 11/12, stretch 0/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
+Seen: 12/16  (must 12/12, stretch 0/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
```

The step in between was `kubectl set image` to a second tag followed by `kubectl rollout undo`. Final result for devboard, with the Rolling update row now earned:

```
$ ./audit.sh
context: kind-audit   scope: all non-system namespaces

✅ Deployment + ReplicaSet              2 deployment(s) in app namespaces
✅ Service                              3 service(s)
✅ Namespace                            1 namespace(s) besides default and system ones
✅ Labels and selectors                 3 service(s) have endpoints, so selectors match pod labels
✅ Rolling update + rollback            1 deployment(s) rolled to a new image (rollback itself is not visible, show it in your evidence)
✅ ConfigMap                            2 non-default configmap(s)
✅ Secret                               1 Opaque secret(s)
✅ Requests and limits                  3 of 3 containers have cpu+memory requests and limits
✅ Probes (liveness+readiness)          3 of 3 containers have both probes
✅ PVC                                  1 bound PVC(s)
✅ Ingress                              1 ingress resource(s), 1 running controller pod(s)
✅ Multi-node cluster                   3 node(s)
❌ HPA (stretch)                        0 HPA(s)
❌ RBAC + ServiceAccount (stretch)      0 role/rolebinding(s), 0 custom serviceaccount(s)
❌ CronJob (stretch)                    0 cronjob(s)
❌ GitHub Actions to kind (stretch)     looks for kind create cluster / helm/kind-action in ./.github/workflows

Seen: 12/16  (must 12/12, stretch 0/4). Pass mark is 10+. Now write the evidence in K8S-AUDIT.md.
```

Known limit: a rollout that only changes an environment variable is not counted. The reason column says what the check really looks at. Nothing else misbehaved on this project: StatefulSet pods are counted for resources and probes, the PVC from a `volumeClaimTemplate` is found, and the repo's ten workflow files are correctly reported as not deploying to kind.

### B6. Could the existing GitHub Actions deploy to kind?

Yes. What is there and what it would need:

- `ci-pipeline.yml` (manual) and `devsecops-pipeline.yml` (push to `main`) call `build-and-push.yml`, which pushes `trainwithshubham/devboard-*:<sha>` to Docker Hub, then `update-manifests.yml` writes the new tag into `k8s/eks` and commits it.
- Add a `deploy-kind` job after `build-and-push`: `helm/kind-action` with `kind/kind-config.yaml`, install ingress-nginx (with the node pin from `kind/README.md`), apply `k8s/kind/`, `kubectl rollout status` on all three workloads, then `curl` the UI and `/api/projects`.
- `update-manifests.yml` only edits `k8s/eks`, so `k8s/kind` would keep `:local`. Rewrite the image tag inside the job with `sed` (do not commit it), the same way that file already does for EKS.
- The images are public on Docker Hub and the runner is amd64, so the job can pull them with no `kind load`. For pull requests, `build-and-push` is skipped, so build locally with `docker/build-push-action` using `load: true` and run `kind load docker-image`. That path needs no secrets.
- `build-and-push.yml` sets no `platforms`, so images are amd64 only. Add `linux/arm64` if learners on Apple silicon should pull them (that was the `ErrImagePull` in B1).
- `devops.yml` is not valid YAML (tabs, and `run:` placed directly under `steps:`), and it triggers on every push. Fix or delete it, or it will show a failed run beside the new job.
- No workflow currently creates a kind cluster, so the GitHub Actions row stays ❌ until this job exists.

I did not run any of this on GitHub. The example workflow in `examples/sample-app-k8s/.github/workflows/kind-deploy.yml` follows the same shape for the sample app. Its steps were run by hand.

## Test C: four more projects, as a learner would

To see whether the repo works for projects it was not built around, I took four public projects. For each I copied `audit.sh`, `kind/` and `K8S-AUDIT.md` in ("Easiest" way in the README), wrote the manifests like a learner, deployed on the same kind cluster, and audited with the namespace argument. The kind cluster was built only from the commands in `kind/README.md`. They worked as written.

| Project | Stack | Result |
|---|---|---|
| docker/awesome-compose `react-express-mongodb` | React, Express, Mongo (StatefulSet) | Works. 12/16, then 13/16 after a rolling update and undo. A StatefulSet volume shows up as a bound PVC. |
| `nginx-flask-mysql` | nginx, Flask, MariaDB, password from a file | Works after two fixes (below). 12/16, then 14/16 after adding RBAC and a CronJob. |
| `react-java-mysql` | React, Spring Boot, MariaDB | Works. 10/16. The ❌ rows were true: no ConfigMap, no rollout, no stretch items. |
| iemafzal/EasyShop | Next.js, Mongo, Redis, with its own `kubernetes/` folder written for EKS | Applied unchanged: Mongo never starts. Ported: works. 11/16, and the audit pointed at a real gap (Redis had no limits). |

What went wrong, and what it means for learners:

- **Secret mounted at `/run/secrets`.** Copying the compose path made the container fail with `read-only file system`, because it clashes with the ServiceAccount token mount. Fix: mount the single file with `subPath`. Added to the README troubleshooting.
- **App broke on its own dependencies.** The Flask project fails in plain Docker too (unpinned `Werkzeug`). Pinning it fixed it. This is why the README says to confirm the app runs with Docker first.
- **A probe that goes through to another service** (nginx `/` to the backend) caused a restart loop while the backend was down. Probe the container itself.
- **Wrong Dockerfile stage.** Several projects build a dev stage in compose with `target:`. A plain `docker build` picks the last stage. Copy the `target`.
- **EasyShop on EKS manifests:** a `gp2` StorageClass and a PV of type `awsElasticBlockStore` with a placeholder volume ID. The PVC showed Bound, but the pod waited forever on `ebs.csi.aws.com ... vol-12345`. Also: a cert-manager issuer and TLS (CRDs not installed), a real hostname, and a kind config and kustomization file sitting in the same folder, which `kubectl apply -f` cannot handle. Fix: use the local-path StorageClass, drop TLS, use `*.localtest.me`, apply only the files you need.
- **Migration jobs run once.** EasyShop's seed job ran while Mongo was down, so the shop had no products. Delete and re-apply the Job.

### audit.sh changes because of Test C

1. **A broken app could score a pass.** EasyShop with its EKS manifests had no running pods, but the audit said 10/16 and "Pass mark is 10+", because the PVC, Ingress, HPA, ConfigMap and Secret objects existed. The script now prints a warning with the pod names when any pod is not Running/Ready.
2. **Init containers were ignored** by the requests/limits check. They are counted now (they cannot have probes, so the probe check still skips them).

Both changes only add output or counts. The sample app and devboard have no init containers and healthy pods, so their results above are unchanged.

I did not finish the last EasyShop step (add Redis limits, rolling update, re-audit). The two fixes it would have shown were already proven on the other projects.

## Other checks

- `shellcheck audit.sh`: clean.
- `audit.sh` exits 0 in every case I tried: a namespace that does not exist, the default namespace, and no reachable cluster (`KUBECONFIG=/nonexistent ./audit.sh` prints one line and exits 0).
- Links: I requested every external link in the repo's files. All returned 200 except local example URLs (`localhost`, `*.localtest.me`) that only work once the app runs.
