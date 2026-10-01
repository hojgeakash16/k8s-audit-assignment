# Sample app: example voting app

Use this if you do not have a project of your own. It is plain Docker only. Writing the Kubernetes manifests is your job.

```
vote (Python/Flask)  -->  redis  -->  worker (.NET)  -->  postgres  <--  result (Node.js)
   web page to vote       queue         moves votes         storage        web page with live results
```

Five pieces: two web frontends, one queue, one worker, one database. The worker has no port and no web page.

The code is copied from [k8s-kind-voting-app](https://github.com/LondheShubham153/k8s-kind-voting-app) (Docker's example voting app, Apache-2.0, see `LICENSE`). Two small changes: `result` and `worker` read the database password from the `POSTGRES_PASSWORD` environment variable (default `postgres`), so a Kubernetes Secret has a real job. The upstream repo also has a `k8s-specifications/` folder. Try the assignment first before you look at it.

## Run with Docker

```bash
cd sample-app
docker compose up --build
```

- Vote: <http://localhost:8081>
- Result: <http://localhost:8082>

Vote a few times and watch the result page change. Stop with `docker compose down -v`.

## Things the app expects (you will need these in Kubernetes)

- `vote` talks to a host called `redis` on port 6379.
- `worker` talks to `redis` and to a host called `db` (Postgres, user `postgres`).
- `result` talks to `db`.
- `vote`, `result` listen on port 80. `vote` also reads `OPTION_A` and `OPTION_B` (the two choices).
- Postgres stores data in `/var/lib/postgresql/data`.

So your Services must be named `redis` and `db`.

## Build images and load them into kind

```bash
docker build -t vote:1.0   ./vote
docker build -t result:1.0 ./result
docker build -t worker:1.0 ./worker
kind load docker-image vote:1.0 result:1.0 worker:1.0 --name audit
```

In your manifests use `image: vote:1.0` with `imagePullPolicy: IfNotPresent`. The worker uses .NET and the first build is slow.
For the rolling update, build or tag a second version (`docker tag vote:1.0 vote:2.0`, change something visible, load it) and run `kubectl set image`.
