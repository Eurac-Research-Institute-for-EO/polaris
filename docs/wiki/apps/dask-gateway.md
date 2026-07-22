# Dask Gateway

[Dask Gateway](https://gateway.dask.org/) lets you run [Dask](https://docs.dask.org/)
clusters **on Polaris instead of on your laptop**. You keep writing normal Python in
your usual notebook or script — the only thing that changes is *where* the work
executes: instead of your 8 cores and 16 GB of RAM, it runs on the cluster's compute
nodes.

It is deployed via ArgoCD (`apps/dask-gateway/` in `polaris-apps`) into the
`dask-gateway` namespace.

!!! warning "Internal demo — no access control"
    The gateway has **no authentication**: anyone on the Eurac network who can reach
    the endpoint can start clusters. Resource caps (below) are the only thing
    stopping one person from taking the whole pool.

## Endpoint

```
http://10.8.244.185:30080
```

Quick check that it's alive:

```bash
curl http://10.8.244.185:30080/api/version
# {"version": "2026.3.0"}
```

## Install the client

```bash
pip install "dask-gateway==2026.3.0" "dask[distributed]==2026.3.0"
# or
conda install -c conda-forge dask-gateway=2026.3.0 dask=2026.3.0
# if you are not stuck in the past:
uv add "dask-gateway==2026.3.0" "dask[distributed]==2026.3.0"
```

!!! important "Version matching"
    The server runs **2026.3.0**. Dask is sensitive to version skew between your
    local client and the remote scheduler/workers — mismatches show up as pickle
    errors or confusing warnings. Pin the same release locally:

    After connecting, `client.get_versions(check=True)` raises if client, scheduler
    and workers disagree.

## Quickstart

```python
from dask_gateway import Gateway

gateway = Gateway("http://10.8.244.185:30080")

cluster = gateway.new_cluster()      # scheduler starts, 0 workers
cluster.scale(2)                     # ask for 2 workers
client = cluster.get_client()        # route all dask work here

print(cluster.dashboard_link)        # watch it live in a browser

# ... your normal dask code ...

client.close()                       # disconnect the local client
cluster.shutdown()                   # stop the workers and the scheduler
```

`new_cluster()` returns as soon as the scheduler is up; workers arrive a few seconds
later as Kubernetes pulls the image and schedules the pods. `client.wait_for_workers(2)`
blocks until they're actually there, which is what you want before timing anything.

!!! tip "Use a context manager so you never leak a cluster"
    Forgetting `shutdown()` is the single most common way the pool fills up. The
    `with` form releases the cluster even if your code raises:

    ```python
    with gateway.new_cluster() as cluster:
        cluster.scale(4)
        client = cluster.get_client()
        result = my_computation.compute()
    # cluster is shut down here
    ```

## Cluster options

The gateway exposes three knobs, configured server-side. Inspect them from Python:

```python
options = gateway.cluster_options()
print(dict(options))
# {'worker_cores': 2, 'worker_memory': 16.0, 'image': 'pangeo/pangeo-notebook:04bb14b'}
```

| Option | Default | Range | Meaning |
| --- | --- | --- | --- |
| `worker_cores` | `2` | 1–8 | CPU cores **per worker** |
| `worker_memory` | `16.0` | 1–64 | GiB of RAM **per worker** |
| `image` | `pangeo/pangeo-notebook:04bb14b` | any pullable image | Container the scheduler and workers run |

Set them when creating the cluster:

```python
cluster = gateway.new_cluster(worker_cores=4, worker_memory=32)
```

In a notebook, `options` renders as an interactive widget you can fill in by hand,
then pass along:

```python
options = gateway.cluster_options()
options                              # displays sliders/fields in Jupyter
cluster = gateway.new_cluster(options)
```

!!! note "One worker must fit on one node"
    A worker is a single pod, so it can never be larger than one node — hence the
    8-core / 64 GiB ceiling. Scale *out* (more workers), not *up*, for bigger jobs.
    Many small workers also schedule faster than a few huge ones, because there are
    more places they fit.

### How many workers can I get?

Per-cluster caps are `cluster_max_cores = 24`, `cluster_max_memory = 192 G`,
`cluster_max_workers = 24`. The scheduler pod counts against them, so the effective
worker count is roughly `(24 - 1) // worker_cores`:

| `worker_cores` | Max workers | Total worker cores |
| --- | --- | --- |
| 1 | 23 | 23 |
| 2 (default) | 11 | 22 |
| 4 | 5 | 20 |
| 8 | 2 | 16 |

Memory caps independently — with `worker_memory=64` you get 2 workers regardless of
cores. `cluster.scale()` above the cap is clamped rather than raising.

These are **per cluster**, not per person. Two colleagues can each run a full-size
cluster and still leave headroom for the openEO workloads. A third simply waits:
pods that don't fit sit `Pending` until capacity frees, they don't evict anything.

## Scaling

**Fixed** — you decide, workers stay until you change it:

```python
cluster.scale(8)
```

**Adaptive** — the scheduler adds and removes workers based on the actual queue:

```python
cluster.adapt(minimum=1, maximum=10)
cluster.adapt()          # turn adaptive mode back off (back to manual scaling)
```

Adaptive is the polite default for interactive/exploratory work: you get capacity
during a `.compute()` and hand it back while you're reading the output. Prefer fixed
scaling for benchmarking, or when your workload has long gaps that would cause
workers to be dropped and re-acquired repeatedly.

!!! danger "Idle clusters are reclaimed after 1 hour"
    A cluster with **no work** for 60 minutes is shut down automatically — this is
    the backstop against abandoned clusters, not a reason to skip `shutdown()`.
    A cluster actively running a graph is never idle and is never touched. Note
    that "idle" means no *tasks*: a notebook holding a big result in worker memory
    with nothing running still counts as idle and **will** be reclaimed, taking that
    data with it. Persist anything you care about to storage.

## Reconnecting to a running cluster

Your clusters survive your Python process dying — kernel restart, closed laptop,
crashed script. List and reattach instead of starting a new one:

```python
gateway.list_clusters()
# [ClusterReport<name=abc123..., status=RUNNING>]

cluster = gateway.connect(gateway.list_clusters()[0].name)
client = cluster.get_client()
```

You only see your own clusters. The client authenticates as your **local OS
username** by default (with an empty password, which the gateway's `simple` auth
accepts), so clusters are namespaced per user without you configuring anything. To
use a different identity explicitly:

```python
from dask_gateway.auth import BasicAuth
gateway = Gateway("http://10.8.244.185:30080", auth=BasicAuth("jzvolensky"))
```

Shut down everything you have running:

```python
for c in gateway.list_clusters():
    gateway.stop_cluster(c.name)
```

## The dashboard

```python
cluster.dashboard_link
# http://10.8.244.185:30080/clusters/<user>.<id>/status
```

It is proxied through the gateway, so the link works from any machine on the network
with no port-forwarding. This is the fastest way to see whether your problem is too
few workers, unbalanced partitions, or memory pressure — the task stream and worker
memory bars tell you within seconds.

## Integrating with your existing code

The point of Dask is that **your analysis code doesn't change**. Creating a `Client`
registers it as the global scheduler, and every subsequent dask operation — `xarray`
opened with `chunks=`, `dask.array`, `dask.dataframe`, `dask.delayed` — runs on the
cluster instead of locally.

```python
import xarray as xr
from dask_gateway import Gateway

gateway = Gateway("http://10.8.244.185:30080")

with gateway.new_cluster(worker_cores=4, worker_memory=32) as cluster:
    cluster.adapt(minimum=2, maximum=8)
    client = cluster.get_client()

    ds = xr.open_dataset("/mnt/CEPH_PROJECTS/my_project/cube.zarr",
                         engine="zarr", chunks={"time": 10})
    monthly = ds["NDVI"].resample(time="1M").mean()
    result = monthly.compute()        # <- runs on Polaris

result.to_netcdf("local_output.nc")   # small result, written locally
```

Embarrassingly parallel loops — the common "run this function over 500 scenes" case —
map cleanly with `delayed` or the futures API:

```python
import dask

@dask.delayed
def process_scene(url):
    ...
    return stats                       # small! see below

results = dask.compute(*[process_scene(u) for u in scene_urls])
```

Three things decide whether this actually works:

### 1. Where your data lives

**The CephFS shares work, with the paths you already use.** Every worker has:

| Path | Access |
| --- | --- |
| `/mnt/CEPH_PROJECTS` | read-write |
| `/mnt/CEPH_BASEDATA` | read-only |
| `/mnt/CEPH_PRODUCTS` | read-only |

They are the same shares mounted on your workstation, so a path that works locally
works on a worker unchanged — this is the main reason offloading is cheap here:

```python
ds = xr.open_mfdataset("/mnt/CEPH_PROJECTS/my_project/cube/*.nc", chunks={"time": 10})
```

### 2. Workers need your libraries

Whatever your tasks import must exist **in the worker image**, or they fail with
`ModuleNotFoundError`. Installing a package locally does nothing for the workers.

The default is [`pangeo/pangeo-notebook`](https://github.com/pangeo-data/pangeo-docker-images),
which covers most of what EO work needs out of the box:

| | Version |
| --- | --- |
| `xarray` | 2026.4.0 |
| `rasterio` / `rioxarray` | 1.5.0 / 0.22.0 |
| `geopandas` / `dask-geopandas` | 1.1.3 / 0.5.0 |
| `zarr` / `s3fs` | 3.2.1 / 2026.4.0 |
| `dask` / `distributed` / `dask-gateway` | 2026.3.0 |

That last row is the important one: it matches the gateway server exactly, so if you
pin `2026.3.0` locally the client, scheduler and workers all agree.

Anything not in that list — `openeo`, an internal package, a specific SAR toolbox —
needs your own image:

```python
cluster = gateway.new_cluster(image="ghcr.io/eurac-research-institute-for-eo/my-dask-env:1.2.0")
```

!!! important "Custom images must have `dask-gateway` installed"
    The scheduler is launched with `--preload dask_gateway.scheduler_preload`, so an
    image without the `dask-gateway` package fails to start the cluster at all.
    Keep `dask`, `distributed` **and** `dask-gateway` at 2026.3.0. Easiest is to
    build on the image the cluster already uses:

    ```dockerfile
    FROM pangeo/pangeo-notebook:04bb14b
    RUN mamba install -y -c conda-forge my-extra-package && mamba clean -afy
    ```

    The image must be pullable by the nodes — public GHCR, or an image pull secret
    in the `dask-gateway` namespace.

!!! note "Custom images are slow the first time"
    The default image is pre-pulled onto every compute node by the
    `dask-image-prepull` DaemonSet, so it costs nothing at cluster start. A **custom**
    image is not: the first cluster using it waits for a multi-GB pull, and workers
    take minutes rather than seconds to appear. Start and worker timeouts are raised
    to 600s to allow for this. The node caches it afterwards, so only the first use
    on each node is slow.

### 3. Bring back results, not data

Everything you `.compute()` travels over the network into your local process. Reduce
on the cluster and return the small thing — statistics, an aggregated time series, a
downsampled array. If the output is large, have the workers write it directly to
storage (`ds.to_zarr("/mnt/CEPH_PROJECTS/...")`) and return nothing.

### When *not* to use it

- **Data smaller than your RAM, code that isn't dask-aware.** Plain numpy/pandas on
  your laptop wins; the cluster adds latency and serialization for nothing.
- **A single long non-parallel task.** One worker will run it, no faster than your
  own machine.
- **Scheduled or reproducible production pipelines.** That's what the openEO / Argo
  Workflows path is for. Dask Gateway is for interactive and ad-hoc analysis.

## Etiquette

Shared pool, no quotas, no authentication — it works on trust (and I do not have any):

1. **Shut down when you're done.** `cluster.shutdown()`, or use `with`.
2. **Use `adapt()` for interactive work** so idle cores go back to the pool.
3. **Ask for what you need.** The default 2 cores / 16 GiB per worker is a sensible
   starting point; scale up after watching the dashboard, not before.
4. **Check `gateway.list_clusters()` before starting another one** — reconnect to the
   one you forgot about instead of stacking a second.

**Note:** Failure to follow these rules will get you a warning. After 3 warnings your laptop will be confiscated and you will be multiplying arrays by hand for the rest of your life.

Happy dasking!

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `scale()` seems capped, or a limit error on create | You hit the 24-core / 192 GB / 24-worker per-cluster limit. Lower `worker_cores` or `worker_memory`. |
| Workers requested but never arrive | Pods are `Pending` — the pool is full, or the request doesn't fit on one node. Check with `kubectl -n dask-gateway get pods`. |
| `ModuleNotFoundError` inside a task | The library isn't in the worker image. See [Workers need your libraries](#2-workers-need-your-libraries). |
| Cluster fails to start with a custom image | The image is missing `dask-gateway`, or the nodes can't pull it. |
| Version-mismatch warnings, odd pickle errors | Client vs worker skew. Pin `2026.3.0`; verify with `client.get_versions(check=True)`. |
| Workers dying mid-computation | OOMKilled — memory limits are enforced. Raise `worker_memory`, or use smaller chunks. |
| Cluster vanished | 1-hour idle timeout. Restart it; persist results next time. |
| Connection refused / hangs | Not on the Eurac network, or the node is down. Try another node IP; test with the `curl` above. |
