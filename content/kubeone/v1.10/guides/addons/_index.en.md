+++
title = "Addons"
date = 2020-04-01T12:00:00+02:00
+++

Addons are a mechanism used to deploy Kubernetes resources after provisioning
the cluster. Addons allow operators to use KubeOne to deploy various components
such as CNI and CCM, and various stacks such as logging and monitoring, backups
and recovery, log rotating, and more.

This document explains how to use addons in your workflow.

## Writing Addons

Addons are represented as Kubernetes YAML manifests. To deploy an addon, the
operator needs to put a YAML manifest in a directory and provide it as the
addons directory in the KubeOne cluster configuration.

### Templating

Manifests support templating based on [Go templates][go-templates].
The following data is available out of the box:

* KubeOne cluster configuration - `.Config`
* Credentials - `.Credentials`

On top of that, you can use the [`sprig`][sprig] functions in your templates.
For list of available functions, consider the [`sprig` docs][sprig-docs].

### Example

The following snippet shows how an addon looks like and how to use templating:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: example-{{ .Config.Name }} # will be rendered as 'example-cluster_name'
---
apiVersion: v1
kind: Secret
metadata:
  name: credentials
  namespace: kube-system
type: Opaque
data:
  AWS_ACCESS_KEY_ID: {{ .Credentials.AWS_ACCESS_KEY_ID | b64enc }} # will be rendered as base64-encoded AWS access key
  AWS_SECRET_ACCESS_KEY: {{ .Credentials.AWS_SECRET_ACCESS_KEY | b64enc }} # will be rendered as base64-encoded AWS secret access key
```

**Note:** The `b64enc` function is a [`sprig` function][sprig-b64enc].

## Addons API Reference

Take a look at the [KubeOne's autogenerated API reference][addons-api] for more
details about available options in the Addons API.

## Enabling Addons

To enable addons, you need to modify the KubeOne cluster configuration to add
the `addons` config:

```yaml
apiVersion: kubeone.k8c.io/v1beta2
kind: KubeOneCluster
versions:
  kubernetes: 1.29.4
cloudProvider:
  aws: {}
# Addons are Kubernetes manifests to be deployed after provisioning the cluster
addons:
  enable: true
  # In case when the relative path is provided, the path is relative
  # to the KubeOne configuration file.
  path: "./addons"
```

The addons path is normalized on the runtime. If you provide a relative path,
the path is relative to the KubeOne configuration file. This means that
`./addons` will be parsed depending on the `kubeone` command you use:
* `kubeone apply -m config.yaml` - `./addons`
* `kubeone apply -m other/dir/config.yaml` - `./other/dir/addons/config.yaml`

{{% notice note %}}
Addons can be organized into subdirectories, but only one level of
subdirectories is supported. For example, `./addons/example-addon-1` is
supported and YAML manifest in that directory will be deployed, but
`./addons/example-addon-1/subdirectory-2` directory will be entirely ignored.
{{% /notice %}}

## Embedded Addons

Since KubeOne v1.3 release, there's a new kind of addons that are always carried
with the binary itself -- embedded addons. This is done thanks to the new Go's
[`embed`][embed-docs] package.

Some of those addons are directly and automatically used by the KubeOne itself
(for example `machinecontroller`), but it's not the case for all of them. Some
very useful addons can be activated by the user to be deployed to the cluster,
like:
* [backups-restic][backups_restic]
* [cluster-autoscaler][cluster_autoscaler]
* [default-storage-class][default_storage_class]
* [unattended-upgrades][unattended_upgrades]

### Activate Embedded Addons

To activate the embedded addons, the user needs to use the new [Addons API][addons-api].

Example:

```yaml
apiVersion: kubeone.k8c.io/v1beta2
kind: KubeOneCluster
versions:
  kubernetes: 1.29.4

addons:
  enable: true
  addons:
  - name: cluster-autoscaler
  - name: unattended-upgrades
```

[Up to date (and possibly unreleased yet) list of addons.][addons-list-url]

### Overriding Embedded Addons

Some of those embedded addons used to be written in Go structures form. Now
KubeOne has them in YAML form so those addons are easier to maintain and to let
users to override them.

For example, if you wish to have your own machine-controller manifests being
deployed to the cluster, you can override the `machinecontroller` addon. Enable
addons like described above and in the addons directory place your
`machinecontroller` manifests in the same directory name, e.g.
`./addons/machinecontroller/yourmanifest.yaml`.

## Cleanup Addons

To delete embedded addon from the cluster, use the new `delete` field from the
[Addons API][addons-api].

```yaml
apiVersion: kubeone.k8c.io/v1beta2
kind: KubeOneCluster
versions:
  kubernetes: 1.29.4

addons:
  enable: true
  addons:
  - name: unattended-upgrades
    delete: true
```

{{% notice note %}}
That will cleanup all the traced of `unattended-upgrades` addon, but only it's
manifests, not the contents that addons workloads might have produced, i.e.
backups themselves.
{{% /notice %}}

{{% notice warning %}}
`delete` field works only on embedded addons for now.
{{% /notice %}}

## Parameters

It's possible to pass down user defined parameters (in key/value form) down to
an addon. There are two fields (of type `map[string]string`):
* `.addons.globalParams`
* `.addons.addons[] .params`

{{% notice note %}}
`.addons.addons[] .params` has higher priority over `.addons.globalParams`, so
you can use it to override globally defined parameters.
{{% /notice %}}

```yaml
apiVersion: kubeone.k8c.io/v1beta2
kind: KubeOneCluster
versions:
  kubernetes: 1.29.4

addons:
  enable: true
  globalParams:
    key1: value1
  addons:
  - name: unattended-upgrades
    params:
      key2: value2
```

### Well-known Parameters

Some embedded addons can be configured via "well-known" parameters. They are documented below.

#### External CCMs

All external CCMs support being configured via parameters (either passed as global parameter
or passed as parameter on the respective CCM addon):

| Parameter     | Description                                                             |
| ------------- | ----------------------------------------------------------------------- |
| `CCM_CONCURRENT_SERVICE_SYNCS` | Sets the parallel `Service` (load balancer) reconciles in the CCM. If not set, this is "1". The value for this parameter is a number, but needs to be wrapped in quotes. **Warning**: Setting this parameter will result in higher CPU and network consumption. It might also trigger rate limits with your cloud provider APIs. Be very conservative about increasing this number. |

## Reconciling Addons

The addons are reconciled upon running `kubeone apply`. If the cluster is
being provisioned for the first time, addons will be applied by the end of
the provisioning process.

```bash
kubeone apply --manifest kubeone.yaml -t .
```

The reconciliation is done using `kubectl` over SSH, using a
command such as:

```
kubectl apply -f addons.yaml --prune -l kubeone.io/addon
```

Using the `--prune` options means that the next time you run `kubeone`:
* if you updated any manifest, the corresponding resources in the cluster will
be updated,
* if you removed a resource from a manifest, the resource will be removed from
the cluster as well
* if you removed a whole manifest, all resources defined in that manifest will
be removed from the cluster

{{% notice warning %}}
The `--prune` option can be **dangerous**. Always make sure that you have all
needed manifests present in the addons directory and correct addons
configuration before running `kubeone`.
{{% /notice %}}

The addons are applied in the alphabetical order. This means that you can
control in which order addons will be applied by setting the
appropriate file name.

## Example Addons

We provide the example addons that you can use as a template or to handle
various tasks, such as cluster backups. You can find the example addons in
the [`addons`][addons] directory.

[go-templates]: https://golang.org/pkg/text/template/
[sprig]: https://github.com/Masterminds/sprig
[sprig-docs]: http://masterminds.github.io/sprig/
[sprig-b64enc]: http://masterminds.github.io/sprig/encoding.html
[addons]: https://github.com/kubermatic/kubeone/tree/release/v1.10/addons
[addons-api]: {{< ref "../../references/kubeone-cluster-v1beta2/#addons" >}}
[embed-docs]: https://pkg.go.dev/embed
[addons-list-url]: https://github.com/kubermatic/kubeone/tree/release/v1.10/addons
[backups_restic]: https://github.com/kubermatic/kubeone/tree/release/v1.10/addons/backups-restic
[cluster_autoscaler]: https://github.com/kubermatic/kubeone/tree/release/v1.10/addons/cluster-autoscaler
[default_storage_class]: https://github.com/kubermatic/kubeone/tree/release/v1.10/addons/default-storage-class
[unattended_upgrades]: https://github.com/kubermatic/kubeone/tree/release/v1.10/addons/unattended-upgrades
