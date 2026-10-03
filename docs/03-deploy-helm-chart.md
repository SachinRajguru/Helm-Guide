
## Deploying Helm Charts

**File:** `03-deploy-helm-chart.md`

## Table of Contents

- [1. Objective](#1-objective)
- [2. Prerequisites](#2-prerequisites)
- [3. Verify Environment](#3-verify-environment)
- [4. Add a Chart Repository](#4-add-a-chart-repository)
- [5. Search for Charts](#5-search-for-charts)
- [6. Install NGINX](#6-install-nginx)
- [7. Customize a Chart](#7-customize-a-chart)
- [8. Render Without Installing](#8-render-without-installing)
- [9. Upgrade NGINX](#9-upgrade-nginx)
- [10. Install Prometheus](#10-install-prometheus)
- [11. Installing Charts From Other Repositories](#11-installing-charts-from-other-repositories)
- [12. AWS Load Balancer Controller](#12-aws-load-balancer-controller)
- [13. Uninstall Releases](#13-uninstall-releases)
- [14. Troubleshooting](#14-troubleshooting)
- [15. Best Practices](#15-best-practices)
- [16. Interview Questions](#16-interview-questions)
- [17. Cleanup](#17-cleanup)

## 1. Objective

This lab demonstrates how to:

- Add a Helm repository.
- Search for charts.
- Install NGINX.
- Inspect chart values.
- Render templates.
- Upgrade a release.
- Search for another application.
- Install Prometheus.
- Add a different vendor repository.
- Understand chart-specific values.
- Install the AWS Load Balancer Controller.
- Uninstall releases.

## 2. Prerequisites

Required:

- Helm
- kubectl
- Running Kubernetes cluster
- Working Kubernetes credentials
- Internet access for public chart repositories

Verify:

```bash
helm version
kubectl version --client
kubectl config current-context
kubectl get nodes
```

## 3. Verify Environment

Always establish the target cluster before installation.

```bash
kubectl config current-context
```

Then:

```bash
kubectl get nodes
```

If multiple clusters exist:

```bash
kubectl config get-contexts
```

Select the intended context:

```bash
kubectl config use-context <context-name>
```

## 4. Add a Chart Repository

Add Bitnami:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Update local repository metadata:

```bash
helm repo update
```

Verify:

```bash
helm repo list
```

A repository entry looks conceptually like:

```text
NAME      URL
bitnami   https://charts.bitnami.com/bitnami
```

> Third-party repositories and chart availability can change over time. Always verify the current vendor documentation and chart versions before production use.

## 5. Search for Charts

Search all charts from Bitnami:

```bash
helm search repo bitnami
```

Search for NGINX:

```bash
helm search repo bitnami | grep -i nginx
```

Search for Prometheus:

```bash
helm search repo bitnami | grep -i prometheus
```

Search for a specific chart:

```bash
helm search repo bitnami/nginx
```

Show all available versions:

```bash
helm search repo bitnami/nginx --versions
```

## 6. Install NGINX

Install:

```bash
helm install my-nginx bitnami/nginx
```

The command follows:

```text
helm install <release-name> <repository>/<chart>
```

Therefore:

```text
my-nginx       = release
bitnami        = repository
nginx          = chart
```

### Verify Release

```bash
helm list
```

Expected conceptual result:

```text
NAME       STATUS     CHART
my-nginx   deployed   nginx-<version>
```

Check status:

```bash
helm status my-nginx
```

Check Kubernetes resources:

```bash
kubectl get pods
kubectl get svc
kubectl get deployments
```

## 7. Customize a Chart

Before customizing a third-party chart, inspect its available values.

```bash
helm show values bitnami/nginx
```

This is extremely important because the chart author determines which values are supported.

For example:

```bash
helm show values bitnami/nginx > nginx-values-reference.yaml
```

Review:

```bash
less nginx-values-reference.yaml
```

or:

```bash
cat nginx-values-reference.yaml
```

### `--set`

A simple override can be provided directly:

```bash
helm upgrade my-nginx bitnami/nginx \
  --set service.type=NodePort
```

The general syntax is:

```bash
--set key=value
```

Nested values use:

```bash
--set parent.child=value
```

For example:

```bash
--set service.type=NodePort
```

### Values File

For repeatable deployments, prefer a values file:

```yaml
service:
  type: NodePort
```

Save as:

```text
nginx-values.yaml
```

Then:

```bash
helm upgrade my-nginx bitnami/nginx \
  --values nginx-values.yaml
```

This is generally easier to review and version-control than long `--set` commands.

## 8. Render Without Installing

One of Helm's most useful debugging commands is:

```bash
helm template my-nginx bitnami/nginx
```

This renders Kubernetes manifests locally.

No release is created.

To save output:

```bash
helm template my-nginx bitnami/nginx \
  > rendered-nginx.yaml
```

You can then inspect:

```bash
less rendered-nginx.yaml
```

### Dry Run

Another useful validation approach:

```bash
helm install my-nginx bitnami/nginx \
  --dry-run \
  --debug
```

This is useful for identifying rendering and configuration problems before an actual installation.

## 9. Upgrade NGINX

Upgrade:

```bash
helm upgrade my-nginx bitnami/nginx \
  --set service.type=NodePort
```

Check:

```bash
helm status my-nginx
```

View history:

```bash
helm history my-nginx
```

Conceptually:

```text
Revision 1
    |
    | helm upgrade
    v
Revision 2
```

### Rollback

If revision 2 introduces a problem:

```bash
helm rollback my-nginx 1
```

Verify:

```bash
helm history my-nginx
helm status my-nginx
```

Rollback is based on the release revision, not the chart name.

## 10. Install Prometheus

Search:

```bash
helm search repo bitnami | grep -i prometheus
```

If the required chart is available and compatible with your environment:

```bash
helm install prometheus bitnami/prometheus
```

Verify:

```bash
helm list
kubectl get pods
```

Depending on the chart version, multiple components may be created.

The key point is that one Helm installation can represent a complex application stack.

## 11. Installing Charts From Other Repositories

Not every application is hosted by Bitnami.

The general process is:

```text
Find vendor documentation
        |
        v
Find official chart repository
        |
        v
helm repo add
        |
        v
helm repo update
        |
        v
helm search repo
        |
        v
helm show values
        |
        v
helm install
```

Never assume a chart exists in a particular repository.

## 12. AWS Load Balancer Controller

The AWS Load Balancer Controller is an important real-world Helm example for Amazon EKS.

AWS currently recommends Helm for new EKS users because it simplifies installation.

Add the AWS EKS chart repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
```

Update:

```bash
helm repo update eks
```

Search:

```bash
helm search repo eks
```

Search specifically:

```bash
helm search repo eks/aws-load-balancer-controller
```

List available versions:

```bash
helm search repo eks/aws-load-balancer-controller --versions
```

AWS's current documentation uses the Helm chart and requires environment-specific values such as the cluster name and, depending on the configuration, region/VPC information.

### Example Installation

The following command demonstrates the structure:

```bash
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0
```

The `1.14.0` version shown above is the version currently documented by AWS for this installation example; in a real environment, verify the currently supported version and prerequisites before deployment.

For configurations requiring region and VPC values:

```bash
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --version <AWS_LBC_HELM_VERSION> \
  --set clusterName=<CLUSTER_NAME> \
  --set region=<AWS_REGION> \
  --set vpcId=<VPC_ID> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

AWS specifically documents this pattern for configurations where these values are required.

### What Does `--set` Mean?

Consider:

```bash
--set clusterName=my-cluster
```

This means:

```text
Chart's values.yaml
        |
        +--> clusterName
        |
        v
my-cluster
```

The chart contains a default `values.yaml`.

Instead of editing that file, Helm allows the user to override values from the command line.

For example:

```bash
--set serviceAccount.create=false
```

means:

```yaml
serviceAccount:
  create: false
```

and:

```bash
--set serviceAccount.name=aws-load-balancer-controller
```

means:

```yaml
serviceAccount:
  name: aws-load-balancer-controller
```

### How Do You Know Which Values Are Available?

Use:

```bash
helm show values eks/aws-load-balancer-controller
```

This displays the chart's default values.

Then determine which values are relevant to your environment.

This is much safer than guessing parameter names.

### CRD Consideration

AWS documents an important operational detail: the initial Helm installation handles the controller's CRDs, while an upgrade does not automatically install updated CRDs.

Therefore, when upgrading a chart containing CRDs:

1. Read the vendor upgrade instructions.
2. Check whether CRDs changed.
3. Apply required CRD updates separately if instructed.
4. Then perform the Helm upgrade.

## 13. Uninstall Releases

Uninstall NGINX:

```bash
helm uninstall my-nginx
```

Verify:

```bash
helm list
kubectl get pods
```

Uninstall Prometheus:

```bash
helm uninstall prometheus
```

The argument is the **release name**.

For example:

```bash
helm install nginxv1 bitnami/nginx
```

must be removed with:

```bash
helm uninstall nginxv1
```

not:

```bash
helm uninstall nginx
```

unless `nginx` was actually the release name.

## 14. Troubleshooting

### Chart not found

Check:

```bash
helm repo list
helm repo update
helm search repo <repository>
```

### Wrong values

Inspect:

```bash
helm show values <repo>/<chart>
```

Render:

```bash
helm template <release> <repo>/<chart>
```

### Pod not starting

Check:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Release failed

Check:

```bash
helm status <release>
helm history <release>
```

Then inspect Kubernetes resources:

```bash
kubectl get all
kubectl get events --sort-by=.lastTimestamp
```

### Wrong Kubernetes cluster

Check:

```bash
kubectl config current-context
```

## 15. Best Practices

### Before installation

```bash
helm repo update
helm show values <repo>/<chart>
helm template <release> <repo>/<chart>
```

### For production

Prefer:

```bash
helm upgrade --install
```

with explicit configuration and version control.

Pin versions:

```bash
--version <chart-version>
```

Avoid:

```text
latest
```

for production container images whenever deterministic deployments matter.

Prefer values files for large configurations:

```bash
-f production-values.yaml
```

Use `--set` for small, explicit overrides.

## 16. Interview Questions

### Q1. How do you install an application using Helm?

```bash
helm repo add <repo> <url>
helm repo update
helm search repo <repo>
helm install <release> <repo>/<chart>
```

### Q2. What is a release?

A release is a deployed instance of a Helm chart.

### Q3. Why do you need a release name?

It uniquely identifies the deployed instance and allows Helm to manage its lifecycle.

### Q4. How do you check chart configuration?

```bash
helm show values <repo>/<chart>
```

### Q5. How do you preview generated YAML?

```bash
helm template <release> <chart>
```

### Q6. How do you upgrade a release?

```bash
helm upgrade <release> <chart>
```

### Q7. How do you roll back?

```bash
helm history <release>
helm rollback <release> <revision>
```

### Q8. What is `--set`?

It overrides Helm values from the command line.

### Q9. How do you troubleshoot a failed Helm deployment?

Start with:

```bash
helm status <release>
helm history <release>
helm get values <release>
helm get manifest <release>
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod>
```

### Q10. What is the advantage of `helm template`?

It allows the generated Kubernetes manifests to be inspected without creating the release.

## 17. Cleanup

### Remove NGINX

```bash
helm uninstall my-nginx
```

### Remove Prometheus

```bash
helm uninstall prometheus
```

### Remove AWS Load Balancer Controller if installed

```bash
helm uninstall aws-load-balancer-controller \
  -n kube-system
```

Follow the AWS cleanup documentation for AWS resources that may have been created through the controller.

### Remove local repositories

```bash
helm repo remove bitnami
helm repo remove eks
```

Verify:

```bash
helm list --all-namespaces
helm repo list
kubectl get pods
```

> Do not uninstall shared production components or remove AWS resources merely because this guide contains the commands. Cleanup should only target resources created specifically for the lab.