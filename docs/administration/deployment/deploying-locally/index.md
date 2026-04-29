# Deploying locally

## For researchers

If you are a researcher and would like to try out deploying a small REANA cluster on your laptop,
you can proceed as follows.

**1.** Install [docker](https://docs.docker.com/engine/install/),
[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/),
[kind](https://kind.sigs.k8s.io/docs/user/quick-start/), and
[helm](https://helm.sh/docs/intro/install/) dependencies.

**2.** Deploy REANA cluster:

```{ .console .copy-to-clipboard }
$ wget https://raw.githubusercontent.com/reanahub/reana/master/etc/kind-localhost-30443.yaml
$ kind create cluster --config kind-localhost-30443.yaml
$ wget https://raw.githubusercontent.com/reanahub/reana/master/scripts/prefetch-images.sh
$ sh prefetch-images.sh
$ wget https://raw.githubusercontent.com/reanahub/reana/master/etc/myvalues.yaml
$ helm repo add reanahub https://reanahub.github.io/reana
$ helm repo update
$ helm install reana reanahub/reana --namespace reana --create-namespace --wait -f myvalues.yaml
```

The `myvalues.yaml` file contains illustrative weak credentials suitable for local
single-user trial deployments. **Do not use these values in production.** Production
deployments must replace every secret with a strong, randomly-generated value; see
[deploying at scale](../deploying-at-scale/index.md) for the recommended setup.

**3.** Create REANA admin user:

```{ .console .copy-to-clipboard }
$ wget https://raw.githubusercontent.com/reanahub/reana/master/scripts/create-admin-user.sh
$ sh create-admin-user.sh reana reana john.doe@example.org mysecretpassword
```

**4.** Log into your REANA instance:

```{ .console .copy-to-clipboard }
$ firefox https://localhost:30443
```

**5.** Follow instructions displayed on the web page to run your first REANA analysis example.

## For developers

If you are a developer and would like to install REANA locally and contribute to the REANA cluster code,
please see the [REANA wiki on GitHub](https://github.com/reanahub/reana/wiki).
