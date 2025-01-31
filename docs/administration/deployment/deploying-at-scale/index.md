# Deploying at scale

REANA can be easily deployed on large Kubernetes clusters consisting of many
nodes. This is useful for production instances with many users and many
concurrent jobs.

## Pre-requisites

- A Kubernetes cluster with version greater than v1.21;
- Helm v3;
- A shared POSIX file system volume (such as CephFS, NFS) to host the REANA
  infrastructure volumes and the user runtime workspaces. The shared file system
  is necessary for any multi-node deployment conditions. See [Configuring storage
  volumes](../../configuration/configuring-storage-volumes).

## Multi-node setup

For a scalable multi-user deployment of REANA, it is essential to use a
Kubernetes cluster consisting of several nodes.

We shall be separating various REANA services into various dedicated nodes in
order to ensure that the user runtime workloads would not interfere with the
REANA infrastructure services that are critical for the platform to operate.

We recommend to start with at least six worker nodes:

```console
$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master-0   Ready    master   97m   v1.18.2
node-0     Ready    <none>   97m   v1.18.2
node-1     Ready    <none>   97m   v1.18.2
node-2     Ready    <none>   97m   v1.18.2
node-3     Ready    <none>   97m   v1.18.2
node-4     Ready    <none>   97m   v1.18.2
node-5     Ready    <none>   97m   v1.18.2
```

The worker node roles are ensured by means of labelling the nodes:

- 1 node labelled `reana.io/system=infrastructure` that will run the REANA
  infrastructure services such as the web interface application, the REST API
  server, and the workflow orchestration controller;

- 1 node labelled `reana.io/system=infrastructuredb` that will run the
  PostgreSQL database service (unless you have some already-existing database
  service running outside of the cluster that could be reused without hosting the
  database yourself; this would be even more preferable);

- 1 node labelled `reana.io/system=infrastructuremq` that will run the RabbitMQ
  messaging service;

- 1 node labelled `reana.io/system=runtimebatch` that will run the user runtime
  batch workflow orchestration pods (such as CWL, Snakemake or Yadage processes);

- 1 node labelled `reana.io/system=runtimejobs` that will run the user runtime
  job workload pods (generated by the above workflow batch orchestration pods);

- 1 node labelled `reana.io/system=runtimesessions` that will run the user
  interactive notebook sessions.

For example, you would label the above cluster nodes as follows:

```bash
kubectl label node node-0 reana.io/system=infrastructure
kubectl label node node-1 reana.io/system=infrastructuredb
kubectl label node node-2 reana.io/system=infrastructuremq
kubectl label node node-3 reana.io/system=runtimebatch
kubectl label node node-4 reana.io/system=runtimejobs
kubectl label node node-5 reana.io/system=runtimesessions
```

You would then configure your REANA deployment by means of the Helm
`myvalues.yaml` file as follows:

```yaml
node_label_infrastructure: reana.io/system=infrastructure
node_label_infrastructuredb: reana.io/system=infrastructuredb
node_label_infrastructuremq: reana.io/system=infrastructuremq
node_label_runtimebatch: reana.io/system=runtimebatch
node_label_runtimejobs: reana.io/system=runtimejobs
node_label_runtimesessions: reana.io/system=runtimesessions
```

## Deployment

You would deploy REANA us usual. Start by adding the REANA chart repository:

```console
$ helm repo add reanahub https://reanahub.github.io/reana
"reanahub" has been added to your repositories
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "reanahub" chart repository
...Successfully got an update from the "cern" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

Continue with deploying REANA using your `myvalues.yaml` Helm values file: (see
the list of [supported Helm
values](https://github.com/reanahub/reana/blob/master/helm/reana/README.md))

```console
$ vim myvalues.yaml  # customise your desired Helm values
$ helm install reana reanahub/reana -f myvalues.yaml --wait
NAME: reana
LAST DEPLOYED: Wed Mar 18 10:27:06 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thanks for flying REANA 🚀
```

!!! warning

    Note that the above `helm install` command used `reana` as the Helm release
    name. You can choose any other name provided that it is less than 13 characters
    long. (This is due to current limitation on the length of generated pod names.)

!!! note

    Note that you can deploy REANA in different namespaces by passing `--namespace`
    to `helm install`. Remember to pass `--create-namespace` if the namespace you
    want to use does not exist yet. For more information on how to work with
    namespaces, please see the [Kubernetes namespace
    documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).

## Scaling up

With the above multi-node deployment scenario, it is easy to scale the cluster
up for running heavier workloads or for welcoming more concurrent users, should
the service evolve it that direction. You would keep the 3 infrastructure nodes
and scale the 2 runtime nodes (1 batch, 1 jobs) as your needs grow.

For example, you could add to the cluster 50 new nodes, 10 for batch and 40 for
jobs, and label these new nodes with `reana.io/system=runtimebatch` and
`reana.io/system=runtimejobs` labels, and REANA would automatically recognise
and use the new nodes for executing user workloads without any further change.

Ditto, if you see that users are preferring to run numerous Jupyter notebook
sessions, you could add new nodes labelled `reana.io/system=runtimesessions`,
and REANA would automatically use them to run Jupyter notebooks for users.

A typical production deployment could therefore look like:

- 1 infrastructure app node (labelled `reana.io/system=infrastructure`)
- 1 infrastructure DB node (labelled `reana.io/system=infrastructuredb`)
- 1 infrastructure RabbitMQ node (labelled `reana.io/system=infrastructuremq`)
- 5 runtime interactive session nodes (labelled `reana.io/system=runtimesessions`)
- 10 runtime batch nodes (labelled `reana.io/system=runtimebatch`)
- 40 runtime job nodes (labelled `reana.io/system=runtimejobs`)

Here, the first three infrastructure role nodes should be kept stable, whilst
the last three runtime role nodes can be added and removed at will, based on
increasing or decreasing user workload.

We have been operating REANA deployments on clusters of the above setup
consisting typically of 50-100 nodes and 500-1000 cores, with occasional tests
using up to 5000 cores.

## Designing cluster node roles

The optimal number of how many cluster nodes you should reserve for runtime
batch workflows, for runtime job workloads, or for runtime notebook sessions
depends on your users and their typical research workflows that the cluster is
running.

For example, assuming a cluster node of `m2.large` flavour, i.e. about 8 CPU
cores and 16 GB memory per node, one such runtime job node can comfortably hold
8 concurrent user jobs at the full speed (since 1 node has 8 CPU cores). (The
batch jobs do not require full CPU, since the workflow orchestration processes
do not consume a lot of CPUs; they mostly launch user jobs and then wait for
their execution.) Hence, 1 such runtime job node could run comfortably 8 user
jobs, should the memory suffice. (If the workflows are not CPU-bound but
memory-bound, then using higher RAM node flavours could be would be necessary.)

Another important consideration is the typical parallelism of the user
workflows. For example, if the nature of the physics workflows that are run the
most on the system is such that 1 workflow typically generates 4 very lengthy
parallel n-tupling jobs that are running for hours, followed by relatively
quicker statistical analysis jobs after them, then the overall job throughput
would be most likely determined by the former n-tupling jobs, and we may expect
1 runtime job node to serve up to 2 workflows only. Hence, if we would like to
run 80 such workflows concurrently, then we would need to have about 40 runtime
job nodes in order to run the user workloads at optimal sustainable full speed.
