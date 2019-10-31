# Kubecf deployment via Minikube/Bazel + Operator/Release + Kubecf/Checkout

The intended audience of this document are developers wishing to
contribute to the Kubecf project.

Here we explain how to deploy Kubecf locally using:

  - Minikube to manage a local Kubernetes cluster.
  - A cf-operator pinned with Bazel.
  - Kubecf built and deployed from the sources in the current checkout.

## Minikube

Minikube is one of several projects enabling the deployment,
management and tear-down of a local Kubernetes cluster.

The Kubecf Bazel workspace contains targets to deploy and/or tear-down
a Minikube-based cluster. Using these has the advantage of using a
specific version of Minikube. On the other side, the reduced
variability of the development environment is a disadvantage as well,
possibly allowing portability issues to slide through.

|Operation  |Command                            |
|---        |---                                |
|Deployment | `bazel run //dev/minikube:start`  |
|Tear-down  | `bazel run //dev/minikube:delete` |

### Attention, Dangers

Minikube edits the Kubernetes configuration file referenced by the
environment variable `KUBECONFIG`, or `~/.kube/config`.

To preserve the original configuration either make a backup of the
relevant file, or change `KUBECONFIG` to a different path specific to
the intended deployment.

### Advanced configuration

The local [Minikube Documentation](../dev/minikube/README.md) explains
the various environment variables which can be used to configure the
resources used by the cluster (CPUs, memory, disk size, etc.) in
detail.

## cf-operator

The [cf-operator] is the underlying generic tool to deploy a (modified)
BOSH deployment like Kubecf for use.

[cf-operator]: https://github.com/cloudfoundry-incubator/cf-operator

It has to be installed in the same kube cluster Kubecf will be deployed to.

### Deployment and Tear-down

The Kubecf Bazel workspace contains targets to deploy and/or tear-down
cf-operator:

|Operation  |Command                               |
|---        |---                                   |
|Deployment | `bazel run //dev/cf_operator:apply`  |
|Tear-down  | `bazel run //dev/cf_operator:delete` |

## Kubecf

With all the prequisites handled by the preceding sections it is now
possible to build and deploy kubecf itself.

### System domain

The main configuration to set for kubecf is its system domain.
For the Minikube foundation we have to specify it as:

```sh
echo "system_domain: $(minikube ip).xip.io" \
  > "$(bazel info workspace)/dev/kubecf/system_domain_values.yaml"
```

### Deployment and Tear-down

The Kubecf Bazel workspace contains targets to deploy and/or tear-down
kubecf from the sources:

|Operation  |Command                          |
|---        |---                              |
|Deployment | `bazel run //dev/kubecf:apply`  |
|Tear-down  | `bazel run //dev/kubecf:delete` |

In this default deployment kubecf is launched without Ingress, and
uses the Diego scheduler.

### Access

To access the cluster after the cf-operator has completed the
deployment and all pods are active invoke:

```sh
cf api --skip-ssl-validation "https://api.$(minikube ip).xip.io"

# Copy the admin cluster password.
acp=$(kubectl get secret \
	      --namespace kubecf kubecf.var-cf-admin-password \
	      -o jsonpath='{.data.password}' \
	      | base64 --decode)

# Use the password from the previous step when requested.
cf auth -u admin -p "${acp}"
```