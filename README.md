# Kubernetes Audit Assignment

## The assignment

1. Pick any multi-tier application (frontend + API + a database or cache, at minimum). Your own project, or the [sample app](sample-app/) in this repo.
2. Run it on a kind cluster.
3. Kubernetize it: write the manifests and get it fully working inside the cluster.
4. Use as many Kubernetes concepts from the checklist below as make sense for your app.
5. Run `./audit.sh` to see which concepts the cluster actually shows. Then fill in [K8S-AUDIT.md](K8S-AUDIT.md) with evidence for each one.
6. Submit your repo link.

The goal is depth of real use, not ticking boxes. A concept has to do a job for your app. If you cannot say in one line why you used it, leave it out and mark it ❌.

**Time:** about 4 to 5 hours of work. You get one week.

**You already know:** Docker, Linux, Git and GitHub Actions, and you have finished the course ([video](https://youtu.be/W04brGNgxN4)). Nothing here re-teaches it. The audit table links back to the course material for each concept.

## What is this, in plain words?

You have learned Kubernetes. Now you prove it on a real application.

**Kubernetes** runs the pieces of an app (website, API, database) as small containers, keeps them alive, connects them and updates them. **kind** runs a small Kubernetes cluster on your own laptop, inside Docker. No cloud account is needed.

This repo is the **measuring tape**, not the project. You bring a project. This repo gives you:

- a ready kind cluster setup (`kind/`),
- a script (`audit.sh`) that looks at your running cluster and tells you which Kubernetes features it can see,
- a form (`K8S-AUDIT.md`) where you write proof for each feature and why you used it,
- a finished example, so you know what "good" looks like.

The point is to show real understanding. A feature only counts if your app needs it and you can show it working.

## How it works (4 steps)

```
 your project           kind cluster              this repo                your submission
 (website + API   -->   (your YAML files   -->    ./audit.sh        -->    K8S-AUDIT.md filled in,
  + database)            running here)            shows what it sees       pushed to your GitHub repo
```

1. **Bring your project.** Any app with a frontend, an API and a database or cache. No project? Use [sample-app/](sample-app/).
2. **Run it on kind.** Create the cluster with `kind/kind-config.yaml`. Write Kubernetes YAML files (manifests) for each piece of your app and apply them with `kubectl apply -f`.
3. **Audit it.** With your app running, run `./audit.sh`. It reads your cluster and prints ✅ or ❌ for each of the 16 concepts.
4. **Write the proof.** Open `K8S-AUDIT.md`. For every concept, add evidence (a link to your YAML file, or pasted command output) and one line on why your app needs it. Push to GitHub and submit the link.

The script is only a hint. If it shows ✅ but you cannot explain why you used the concept, that row gets no points. If it shows ❌ because of a quirk but you can prove the concept, explain it in the row.

## Using this repo with your own project

Pick one way:

- **Easiest:** copy `audit.sh`, `kind/` and `K8S-AUDIT.md` into your project repo. Run `./audit.sh` from there. Submit that repo.
- **Keep them separate:** clone this repo next to your project and run `REPO=../my-project ./audit.sh` from here (`REPO` only tells the script where your `.github/workflows` folder is). Copy the filled `K8S-AUDIT.md` into your project repo before you submit.

The script audits whatever cluster `kubectl` is pointing at. Its first line prints the context name. Check that it says your kind cluster (`kind-audit`), not some other cluster.

## Example: what you will see

Real output for the sample app, after it was written up for Kubernetes:

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

Read it like this: ✅ means the cluster shows the concept, and the text after it says what was seen (for example "6 of 7 containers have both probes"). The one ❌ is GitHub Actions, because that workflow file was not in the folder being checked.

Then the learner fills one row per concept in `K8S-AUDIT.md`. Here is one good row, taken from the [full filled example](examples/K8S-AUDIT.filled-example.md):

| Concept | Status | Evidence | Why I used it in my app | Where to look |
|---|---|---|---|---|
| PVC | ✅ | [db-data](examples/sample-app-k8s/02-data-tier.yaml#L1-L12). I deleted the Postgres pod and the vote count was still 3. | Votes must survive a Postgres pod restart. | [PVC example](https://github.com/LondheShubham153/kubestarter/blob/main/PersistentVolumes/PersistentVolumeClaim.yaml) |

What makes it good: a link to the real file, proof that it works, and a reason about the app, not about Kubernetes in general. A weak row would say "PVC used for storage" and nothing else.

## Your to-do list

Copy this into your own repo's README or issue and tick it off.

**Setup**
- [ ] Choose your app (your own, or `sample-app/`) and confirm it runs with plain Docker
- [ ] Create the 3-node cluster from `kind/kind-config.yaml`
- [ ] Install ingress-nginx and metrics-server (`kind/README.md`)
- [ ] Build your images and `kind load docker-image` them (use real tags, not `latest`)

**Write the manifests (one concept at a time, check it works before moving on)**
- [ ] Namespace, then a Deployment and a Service for every tier
- [ ] Labels and selectors that match, so each Service has endpoints
- [ ] ConfigMap for settings and Secret for passwords, read by the pods
- [ ] Requests and limits on every container
- [ ] Liveness and readiness probes
- [ ] PVC for the database or any data that must survive a restart
- [ ] Ingress so you can open the app in a browser
- [ ] A rolling update to a new image tag, then a rollback (`kubectl rollout undo`)
- [ ] Stretch: HPA, RBAC + ServiceAccount, CronJob, GitHub Actions deploying to kind

**Prove it**
- [ ] The app works end to end (you can use it, and data survives deleting the database pod)
- [ ] Run `./audit.sh` and read every ❌
- [ ] Fill in every row of `K8S-AUDIT.md`: status, evidence, one line on why
- [ ] The "My app" section lets a stranger run it from a fresh machine
- [ ] Push to a public GitHub repo and submit the link

## The checklist (16 concepts)

Must-have (12):

| # | Concept | # | Concept |
|---|---|---|---|
| 1 | Deployment + ReplicaSet | 7 | Secret |
| 2 | Service | 8 | Requests and limits |
| 3 | Namespace | 9 | Probes (liveness + readiness) |
| 4 | Labels and selectors | 10 | PVC |
| 5 | Rolling update + rollback | 11 | Ingress |
| 6 | ConfigMap | 12 | Multi-node kind cluster |

Stretch (4): HPA, RBAC + ServiceAccount, CronJob, GitHub Actions deploying to kind.

Left out on purpose: NetworkPolicy, PodDisruptionBudget, StatefulSet, taints and tolerations.

## Start here

```bash
kind create cluster --config kind/kind-config.yaml      # 3 nodes
# then install ingress-nginx and metrics-server: kind/README.md
```

- [kind/README.md](kind/README.md): create, delete, ingress-nginx, metrics-server, loading local images
- [sample-app/](sample-app/): a voting app (5 pieces), Docker only, not kubernetized
- [K8S-AUDIT.md](K8S-AUDIT.md): the file you fill in
- [examples/K8S-AUDIT.filled-example.md](examples/K8S-AUDIT.filled-example.md): a finished audit for the sample app, so you can see what a good row looks like. It links to the full set of manifests, so try the sample app on your own first.
- [TESTING.md](TESTING.md): how this repo was tested, with real `audit.sh` output

> **kind quirks**
> - A `LoadBalancer` Service stays `<pending>` on kind unless you add MetalLB. Use Ingress or `kubectl port-forward`.
> - Ingress needs `extraPortMappings` when the cluster is created (it cannot be added later), plus ingress-nginx.
> - metrics-server needs `--kubelet-insecure-tls` on kind, or the HPA shows `<unknown>`.
> - NetworkPolicy is not enforced by kind's default CNI, so you cannot test it here.
> - Local images need `kind load docker-image`, and a tag that is not `latest`.

## My pod will not start

Look at it in this order:

```bash
kubectl -n <ns> get pods
kubectl -n <ns> describe pod <pod>         # read the Events at the bottom
kubectl -n <ns> logs <pod> --previous      # why the last run died
```

| You see | Usual cause |
|---|---|
| `ErrImagePull` / `ImagePullBackOff` | Image not loaded into kind (`kind load docker-image`), tag is `latest`, or the image is for another CPU type |
| `CrashLoopBackOff` and the logs show a connection error | The app started before its database. Use a readiness probe, an init container that waits, or just let it restart |
| `read-only file system` when starting, and you mounted a Secret at `/run/secrets` | Mount only the file: `mountPath: /run/secrets/name` with `subPath: name` |
| Pod stays `Pending` | `describe` shows why: no CPU/memory left, or a PVC that is not bound |
| Keeps restarting and the probe fails | The probe path calls another service that is down, or `initialDelaySeconds` is too short |
| PVC says `Bound` but the pod waits on `AttachVolume` | The PV is cloud-specific (for example `awsElasticBlockStore`). Delete it and use the default StorageClass |
| App works in Docker but not here | Check the build `target:` in your compose file, and that the service names match what your code connects to (`db`, `redis`, ...) |

If `audit.sh` prints a warning that pods are not Ready, fix that first. The ✅ rows only show that objects exist.

## audit.sh

```bash
./audit.sh              # all non-system namespaces
./audit.sh my-namespace # one namespace
REPO=../my-app ./audit.sh   # point the GitHub Actions check at your app repo
```

It uses only `kubectl` and prints ✅ or ❌ for each of the 16 concepts, with a one-line reason. It is a hint, not a grader. It cannot see why you used something, and a few checks are loose on purpose (for example, one Service with endpoints is enough for "labels and selectors"). The evidence you write in `K8S-AUDIT.md` is what gets scored. It also never fails: a missing concept just shows ❌. If any pod is not Running/Ready it prints a warning, because then the app is not really working.

## How to submit

1. Put your app code, manifests, and a filled-in `K8S-AUDIT.md` in one public GitHub repo. Copy `audit.sh` and `kind/` into it too if you want. A reviewer must be able to start from the "My app" section and reproduce it on a new kind cluster.
2. Send the repo link. <!-- TODO: add where to submit (form / Discord channel / deadline) -->

## Scoring

- 1 point for each must-have concept marked ✅ with real evidence (12 points).
- 1 bonus point for each stretch concept marked ✅ with real evidence (4 points).
- Total is out of 16. **Pass at 10 or more.**
- ⚠️ and ❌ score 0. A ✅ without evidence scores 0.
- No peer review in batch 1.

Evidence is a link to a file in your repo, or pasted command output. No screenshots. Each row also needs one line on why you used the concept.

## Adapting an EKS project to kind

Many projects come with manifests written for EKS. They will not run on kind as they are. This section comes from porting [devboard-starter](https://github.com/LondheShubham153/devboard-starter) (`k8s/eks`) to kind. The full notes are in [TESTING.md](TESTING.md) and the ported files are in [examples/devboard-on-kind/](examples/devboard-on-kind/).

Work in a separate folder (`k8s/kind`) and keep the EKS files unchanged. Apply them to kind once, unmodified, and read the errors. That tells you exactly what to change.

| On EKS | What happens on kind | What to do |
|---|---|---|
| StorageClass with the EBS CSI driver (`ebs.csi.aws.com`, gp2/gp3) | PVC stays `Pending` | Delete the StorageClass and the `storageClassName` line. kind's default `standard` class (local-path) creates the volume. |
| Service or Gateway of type `LoadBalancer` with `aws-load-balancer-*` annotations | Stays `<pending>` forever | Remove the annotations. Use a `ClusterIP` Service plus an Ingress with `ingressClassName: nginx`. |
| Gateway API / Envoy Gateway objects | `no matches for kind ...` (the CRDs are not installed) | Replace with one plain `Ingress` for ingress-nginx. For a hostname without editing `/etc/hosts`, use `name.localtest.me` (it points to 127.0.0.1). |
| Images from a registry (Docker Hub, ECR) | `ErrImagePull` | Build locally, `kind load docker-image`, and set `imagePullPolicy: IfNotPresent`. |
| CI-built images that are amd64 only | `no match for platform in manifest` on Apple-silicon Macs | Build the images yourself on your own machine so they match its CPU. |

Not in devboard, and not tested by me, but the same idea: an ALB Ingress (`ingressClassName: alb`) never gets an address on kind, so switch it to `nginx`. IAM role annotations on ServiceAccounts (IRSA) do nothing on kind, so delete the annotation. ECR paths work the same as any other registry path, so build and load locally instead.

Things that already work as they are: Namespaces, ConfigMaps, Secrets, Deployments, StatefulSets, Services of type `ClusterIP`, probes, resources.

After the port, run `./audit.sh`. Treat the ❌ rows as your to-do list. For devboard the port alone scored 9 of 16. Adding probes and resources made it 11, and a real image update plus rollback made it 12.

## Resources

- Course video: <https://youtu.be/W04brGNgxN4>
- [kubestarter](https://github.com/LondheShubham153/kubestarter): concept index and manifests. We link into it and do not change it.
- [kubernetes-in-one-shot](https://github.com/LondheShubham153/kubernetes-in-one-shot): commands by topic, more manifests.
- [kind: configuration](https://kind.sigs.k8s.io/docs/user/configuration/), [kind: ingress](https://kind.sigs.k8s.io/docs/user/ingress/)
- Kubernetes docs: [concepts](https://kubernetes.io/docs/concepts/), [tasks](https://kubernetes.io/docs/tasks/)

The sample app code comes from Docker's example voting app (Apache-2.0), see [sample-app/LICENSE](sample-app/LICENSE).
