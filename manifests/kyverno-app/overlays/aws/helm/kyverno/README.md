# kyverno

Kubernetes Native Policy Management

![Version: 3.4.4](https://img.shields.io/badge/Version-3.4.4-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: v1.14.4](https://img.shields.io/badge/AppVersion-v1.14.4-informational?style=flat-square)

## About

[Kyverno](https://kyverno.io) is a Kubernetes Native Policy Management engine.

It allows you to:
- Manage policies as Kubernetes resources (no new language required.)
- Validate, mutate, and generate resource configurations.
- Select resources based on labels and wildcards.
- View policy enforcement as events.
- Scan existing resources for violations.

This chart bootstraps a Kyverno deployment on a [Kubernetes](http://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

Access the complete user documentation and guides at: https://kyverno.io.

## Installing the Chart

**IMPORTANT IMPORTANT IMPORTANT IMPORTANT**

This chart changed significantly between `v2` and `v3`. If you are upgrading from `v2`, please read `Migrating from v2 to v3` section.

**Add the Kyverno Helm repository:**

```console
$ helm repo add kyverno https://kyverno.github.io/kyverno/
```

**Create a namespace:**

You can install Kyverno in any namespace. The examples use `kyverno` as the namespace.

```console
$ kubectl create namespace kyverno
```

**Install the Kyverno chart:**

```console
$ helm install kyverno --namespace kyverno kyverno/kyverno
```

The command deploys Kyverno on the Kubernetes cluster with default configuration. The [installation](https://kyverno.io/docs/installation/) guide lists the parameters that can be configured during installation.

The Kyverno ClusterRole/ClusterRoleBinding that manages webhook configurations must have the suffix `:webhook`. Ex., `*:webhook` or `kyverno:webhook`.
Other ClusterRole/ClusterRoleBinding names are configurable.

**Notes on using ArgoCD:**

When deploying this chart with ArgoCD you will need to enable `Replace` in the `syncOptions`, and you probably want to ignore diff in aggregated cluster roles.

You can do so by following instructions in these pages of ArgoCD documentation:
- [Enable Replace in the syncOptions](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#replace-resource-instead-of-applying-changes)
- [Ignore diff in aggregated cluster roles](https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/#ignoring-rbac-changes-made-by-aggregateroles)

ArgoCD uses helm only for templating but applies the results with `kubectl`.

Unfortunately `kubectl` adds metadata that will cross the limit allowed by Kubernetes. Using `Replace` overcomes this limitation.

Another option is to use server side apply, this will be supported in ArgoCD v2.5.

Finally, we introduced new CRDs in 1.8 to manage resource-level reports. Those reports are associated with parent resources using an `ownerReference` object.

As a consequence, ArgoCD will show those reports in the UI, but as they are managed dynamically by Kyverno it can pollute your dashboard.

You can tell ArgoCD to ignore reports globally by adding them under the `resource.exclusions` stanza in the ArgoCD ConfigMap.

```yaml
    resource.exclusions: |
      - apiGroups:
          - kyverno.io
        kinds:
          - AdmissionReport
          - BackgroundScanReport
          - ClusterAdmissionReport
          - ClusterBackgroundScanReport
        clusters:
          - '*'
```

Below is an example of ArgoCD Application manifest that should work with this chart.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno
  namespace: argocd
spec:
  destination:
    namespace: kyverno
    server: https://kubernetes.default.svc
  project: default
  source:
    chart: kyverno
    repoURL: https://kyverno.github.io/kyverno
    targetRevision: 2.6.0
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - Replace=true
```

**Notes on using Azure Kubernetes Service (AKS):**

AKS contains a component known as [Admission Enforcer](https://learn.microsoft.com/en-us/azure/aks/faq#can-admission-controller-webhooks-impact-kube-system-and-internal-aks-namespaces) which will attempt to modify Kyverno's webhooks if not excluded explicitly during Helm installation. If Admissions Enforcer is not disabled, this can lead to several symptoms such as high observed CPU usage and potentially cluster instability. Please see the Kyverno documentation [here](https://kyverno.io/docs/installation/platform-notes/#notes-for-aks-users) for more information and how to set this annotation on webhooks.

## Migrating from v2 to v3

Direct upgrades from v2 of the Helm chart to v3 are not supported due to the number of breaking changes and manual intervention is required. Review and select an option after carefully reading below. Because either method requires down time, an upgrade should only be performed during a maintenance window. Regardless of the chosen option, please read all release notes very carefully to understand the full extent of changes brought by Kyverno 1.10. Release notes can be found at https://github.com/kyverno/kyverno/releases.

**IMPORTANT NOTE**: If you currently use [clone-type](https://kyverno.io/docs/writing-policies/generate/#clone-source) generate rules with synchronization enabled, please do not upgrade to 1.10.0 as there is a bug which may prevent synchronization from occurring on all downstream (generated) resources when the source is updated. Please wait for a future patch where this should be resolved. See [issue 7170](https://github.com/kyverno/kyverno/issues/7170) for further details.

### Option 1 - Uninstallation and Reinstallation

The first option for upgrading, which is the recommended option, involves backing up Kyverno policy resources, uninstalling Kyverno, and reinstalling with v3 of the chart. Policy Reports for policies which have background mode enabled will be regenerated upon the next scan interval.

**Pros**

* Reduced complexity with minimal effort
* Allows re-checking older policies against new validation webhooks in 1.10

**Cons**

* Policy Reports which contained results only from admission mode and from policies/rules where background scans were disabled will be lost.
* Requires additional steps if data-type generate rules are used

Follow the procedure below.

1. READ THE COMPLETE RELEASE NOTES FIRST
2. Backup and export all Kyverno policy resources to a YAML manifest. Use the command `kubectl get pol,cpol,cleanpol,ccleanpol,polex -A -o yaml > kyvernobackup.yaml`.
   1. Before performing this step, if you use [data-type](https://kyverno.io/docs/writing-policies/generate/#data-source) generate rules with synchronization enabled (`generate.synchronize: true`) disable synchronization first (set `generate.synchronize: false`). If you do not perform this step first, uninstallation of Kyverno in the subsequent step, which removes all policies, will result in deletion of generated resources.
3. Uninstall your current version of Kyverno.
4. Review the [New Chart Values](#new-chart-values) section and translate your desired features and configurations to the new format.
5. Install the v3 chart with Kyverno 1.10.
6. Restore your Kyverno policies. Use the command `kubectl create -f kyvernobackup.yaml`.
   1. Before performing this step, if step 2.1 applied to you, enable synchronization (set `generate.synchronize: true`) AND add the field `spec.generateExisting: true`. This will cause existing, generated resources to be refreshed with the new labeling system used by Kyverno 1.10. Note that this may increment the `resourceVersion` field on all downstream resources. Also, understand that when re-installing these policies with `spec.generateExisting: true`, it could result in additional resources being created at that moment based upon the current match defined in the policy. You may need to further refine the match/exclude blocks of your rules to account for this.

### Option 2 - Scale to Zero

In the second option, Kyverno policies do not have to be backed up however you perform more manual work in order to prepare for the upgrade to chart v3.

**Pros**

* Policy Reports which contained results from admission mode will be preserved
* Kyverno policies do not need to be backed up first

**Cons**

* Older policies will not be revalidated for correctness according to the breaking schema changes. Some policies may not work as they did before.
* Requires additional steps if data-type generate rules are used

Follow the procedure below.

1. READ THE COMPLETE RELEASE NOTES FIRST
2. Scale the `kyverno` Deployment to zero replicas.
3. If coming from 1.9 and you have installed the cleanup controller, scale the `kyverno-cleanup-controller` Deployment to zero replicas.
4. If step 3 applied to you, now delete the cleanup Deployment.
5. Review the [New Chart Values](#new-chart-values) section and translate your desired features and configurations to the new format.
6. Upgrade to the v3 chart by passing the mandatory flag `upgrade.fromV2=true`.
7. If you use [data-type](https://kyverno.io/docs/writing-policies/generate/#data-source) generate rules with synchronization enabled (`generate.synchronize: true`), after the upgrade modify those policies to add the field `spec.generateExisting: true`. This will cause existing, generated resources to be refreshed with the new labeling system used by Kyverno 1.10. Note that this may increment the `resourceVersion` field on all downstream resources. Also, understand that when making this modification, it could result in additional resources being created at that moment based upon the current match defined in the policy. You may need to further refine the match/exclude blocks of your rules to account for this.

### New Chart Values

In `v3` chart values changed significantly, please read the instructions below to migrate your values:

- `config.metricsConfig` is now `metricsConfig`
- `resourceFiltersExcludeNamespaces` has been replaced with `config.resourceFiltersExcludeNamespaces`
- `excludeKyvernoNamespace` has been replaced with `config.excludeKyvernoNamespace`
- `config.existingConfig` has been replaced with `config.create` and `config.name` to __support bring your own config__
- `config.existingMetricsConfig` has been replaced with `metricsConfig.create` and `metricsConfig.name` to __support bring your own config__
- `namespace` has been renamed `namespaceOverride`
- `installCRDs` has been replaced with `crds.install`
- `testImage` has been replaced with `test.image`
- `testResources` has been replaced with `test.resources`
- `testSecurityContext` has been replaced with `test.securityContext`
- `replicaCount` has been replaced with `admissionController.replicas`
- `updateStrategy` has been replaced with `admissionController.updateStrategy`
- `priorityClassName` has been replaced with `admissionController.priorityClassName`
- `hostNetwork` has been replaced with `admissionController.hostNetwork`
- `dnsPolicy` has been replaced with `admissionController.dnsPolicy`
- `nodeSelector` has been replaced with `admissionController.nodeSelector`
- `tolerations` has been replaced with `admissionController.tolerations`
- `topologySpreadConstraints` has been replaced with `admissionController.topologySpreadConstraints`
- `podDisruptionBudget` has been replaced with `admissionController.podDisruptionBudget`
- `antiAffinity` has been replaced with `admissionController.antiAffinity`
- `antiAffinity.enable` has been replaced with `admissionController.antiAffinity.enabled`
- `podAntiAffinity` has been replaced with `admissionController.podAntiAffinity`
- `podAffinity` has been replaced with `admissionController.podAffinity`
- `nodeAffinity` has been replaced with `admissionController.nodeAffinity`
- `startupProbe` has been replaced with `admissionController.startupProbe`
- `livenessProbe` has been replaced with `admissionController.livenessProbe`
- `readinessProbe` has been replaced with `admissionController.readinessProbe`
- `createSelfSignedCert` has been replaced with `admissionController.createSelfSignedCert`
- `serviceMonitor` has been replaced with `admissionController.serviceMonitor`
- `podSecurityContext` has been replaced with `admissionController.podSecurityContext`
- `tufRootMountPath` has been replaced with `admissionController.tufRootMountPath`
- `sigstoreVolume` has been replaced with `admissionController.sigstoreVolume`
- `initImage` has been replaced with `admissionController.initContainer.image`
- `initResources` has been replaced with `admissionController.initContainer.resources`
- `image` has been replaced with `admissionController.container.image`
- `image.pullSecrets` has been replaced with `admissionController.imagePullSecrets`
- `resources` has been replaced with `admissionController.container.resources`
- `service` has been replaced with `admissionController.service`
- `metricsService` has been replaced with `admissionController.metricsService`
- `initContainer.extraArgs` has been replaced with `admissionController.initContainer.extraArgs`
- `envVarsInit` has been replaced with `admissionController.initContainer.extraEnvVars`
- `envVars` has been replaced with `admissionController.container.extraEnvVars`
- `extraArgs` has been replaced with `admissionController.container.extraArgs`
- `extraInitContainers` has been replaced with `admissionController.extraInitContainers`
- `extraContainers` has been replaced with `admissionController.extraContainers`
- `podLabels` has been replaced with `admissionController.podLabels`
- `podAnnotations` has been replaced with `admissionController.podAnnotations`
- `securityContext` has been replaced with `admissionController.container.securityContext` and `admissionController.initContainer.securityContext`
- `rbac` has been replaced with `admissionController.rbac`
- `generatecontrollerExtraResources` has been replaced with `admissionController.rbac.clusterRole.extraResources`
- `networkPolicy` has been replaced with `admissionController.networkPolicy`
- all `extraArgs` now use objects instead of arrays
- logging, tracing and metering are now configured using `*Controller.logging`, `*Controller.tracing` and `*Controller.metering`

- Labels and selectors have been reworked and due to immutability, upgrading from `v2` to `v3` is going to be rejected. The easiest solution is to uninstall `v2` and reinstall `v3` once values have been adapted to the changes described above.

- Image tags are now validated and must be strings, if you use image tags in the `1.35` form please add quotes around the tag value.

- Image references are now using the `registry` setting, if you override the registry or repository fields please use `registry` (`--set image.registry=ghcr.io --set image.repository=kyverno/kyverno` instead of `--set image.repository=ghcr.io/kyverno/kyverno`).

- Admission controller `Deployment` name changed from `kyverno` to `kyverno-admission-controller`.
- `config.excludeUsername` was renamed to `config.excludeUsernames`
- `config.excludeGroupRole` was renamed to `config.excludeGroups`

Hardcoded defaults for `config.excludeGroups` and `config.excludeUsernames` have been removed, please review those fields if you provide your own exclusions.

## Uninstalling the Chart

To uninstall/delete the `kyverno` deployment:

```console
$ helm delete -n kyverno kyverno
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Values

The chart values are organised per component.

### Custom resource definitions

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| crds.install | bool | `true` | Whether to have Helm install the Kyverno CRDs, if the CRDs are not installed by Helm, they must be added before policies can be created |
| crds.reportsServer.enabled | bool | `false` | Kyverno reports-server is used in your cluster |
| crds.groups.kyverno | object | `{"cleanuppolicies":true,"clustercleanuppolicies":true,"clusterpolicies":true,"globalcontextentries":true,"policies":true,"policyexceptions":true,"updaterequests":true,"validatingpolicies":true}` | Install CRDs in group `kyverno.io` |
| crds.groups.policies | object | `{"imagevalidatingpolicies":true,"policyexceptions":true,"validatingpolicies":true}` | Install CRDs in group `policies.kyverno.io` |
| crds.groups.reports | object | `{"clusterephemeralreports":true,"ephemeralreports":true}` | Install CRDs in group `reports.kyverno.io` |
| crds.groups.wgpolicyk8s | object | `{"clusterpolicyreports":true,"policyreports":true}` | Install CRDs in group `wgpolicyk8s.io` |
| crds.annotations | object | `{}` | Additional CRDs annotations |
| crds.customLabels | object | `{}` | Additional CRDs labels |
| crds.migration.enabled | bool | `true` | Enable CRDs migration using helm post upgrade hook |
| crds.migration.resources | list | `["cleanuppolicies.kyverno.io","clustercleanuppolicies.kyverno.io","clusterpolicies.kyverno.io","globalcontextentries.kyverno.io","policies.kyverno.io","policyexceptions.kyverno.io","updaterequests.kyverno.io"]` | Resources to migrate |
| crds.migration.image.registry | string | `nil` | Image registry |
| crds.migration.image.defaultRegistry | string | `"reg.kyverno.io"` |  |
| crds.migration.image.repository | string | `"kyverno/kyverno-cli"` | Image repository |
| crds.migration.image.tag | string | `nil` | Image tag Defaults to appVersion in Chart.yaml if omitted |
| crds.migration.image.pullPolicy | string | `"IfNotPresent"` | Image pull policy |
| crds.migration.imagePullSecrets | list | `[]` | Image pull secrets |
| crds.migration.podSecurityContext | object | `{}` | Security context for the pod |
| crds.migration.nodeSelector | object | `{}` | Node labels for pod assignment |
| crds.migration.tolerations | list | `[]` | List of node taints to tolerate |
| crds.migration.podAntiAffinity | object | `{}` | Pod anti affinity constraints. |
| crds.migration.podAffinity | object | `{}` | Pod affinity constraints. |
| crds.migration.podLabels | object | `{}` | Pod labels. |
| crds.migration.podAnnotations | object | `{}` | Pod annotations. |
| crds.migration.nodeAffinity | object | `{}` | Node affinity constraints. |
| crds.migration.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsGroup":65534,"runAsNonRoot":true,"runAsUser":65534,"seccompProfile":{"type":"RuntimeDefault"}}` | Security context for the hook containers |
| crds.migration.podResources.limits | object | `{"cpu":"100m","memory":"256Mi"}` | Pod resource limits |
| crds.migration.podResources.requests | object | `{"cpu":"10m","memory":"64Mi"}` | Pod resource requests |

### Config

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| config.create | bool | `true` | Create the configmap. |
| config.preserve | bool | `true` | Preserve the configmap settings during upgrade. |
| config.name | string | `nil` | The configmap name (required if `create` is `false`). |
| config.annotations | object | `{}` | Additional annotations to add to the configmap. |
| config.enableDefaultRegistryMutation | bool | `true` | Enable registry mutation for container images. Enabled by default. |
| config.defaultRegistry | string | `"docker.io"` | The registry hostname used for the image mutation. |
| config.excludeGroups | list | `["system:nodes"]` | Exclude groups |
| config.excludeUsernames | list | `[]` | Exclude usernames |
| config.excludeRoles | list | `[]` | Exclude roles |
| config.excludeClusterRoles | list | `[]` | Exclude roles |
| config.generateSuccessEvents | bool | `false` | Generate success events. |
| config.resourceFilters | list | See [values.yaml](values.yaml) | Resource types to be skipped by the Kyverno policy engine. Make sure to surround each entry in quotes so that it doesn't get parsed as a nested YAML list. These are joined together without spaces, run through `tpl`, and the result is set in the config map. |
| config.updateRequestThreshold | int | `1000` | Sets the threshold for the total number of UpdateRequests generated for mutateExisitng and generate policies. |
| config.webhooks | object | `{"namespaceSelector":{"matchExpressions":[{"key":"kubernetes.io/metadata.name","operator":"NotIn","values":["kube-system"]}]}}` | Defines the `namespaceSelector`/`objectSelector` in the webhook configurations. The Kyverno namespace is excluded if `excludeKyvernoNamespace` is `true` (default) |
| config.webhookAnnotations | object | `{"admissions.enforcer/disabled":"true"}` | Defines annotations to set on webhook configurations. |
| config.webhookLabels | object | `{}` | Defines labels to set on webhook configurations. |
| config.matchConditions | list | `[]` | Defines match conditions to set on webhook configurations (requires Kubernetes 1.27+). |
| config.excludeKyvernoNamespace | bool | `true` | Exclude Kyverno namespace Determines if default Kyverno namespace exclusion is enabled for webhooks and resourceFilters |
| config.resourceFiltersExcludeNamespaces | list | `[]` | resourceFilter namespace exclude Namespaces to exclude from the default resourceFilters |
| config.resourceFiltersExclude | list | `[]` | resourceFilters exclude list Items to exclude from config.resourceFilters |
| config.resourceFiltersIncludeNamespaces | list | `[]` | resourceFilter namespace include Namespaces to include to the default resourceFilters |
| config.resourceFiltersInclude | list | `[]` | resourceFilters include list Items to include to config.resourceFilters |

### Metrics config

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| metricsConfig.create | bool | `true` | Create the configmap. |
| metricsConfig.name | string | `nil` | The configmap name (required if `create` is `false`). |
| metricsConfig.annotations | object | `{}` | Additional annotations to add to the configmap. |
| metricsConfig.namespaces.include | list | `[]` | List of namespaces to capture metrics for. |
| metricsConfig.namespaces.exclude | list | `[]` | list of namespaces to NOT capture metrics for. |
| metricsConfig.metricsRefreshInterval | string | `nil` | Rate at which metrics should reset so as to clean up the memory footprint of kyverno metrics, if you might be expecting high memory footprint of Kyverno's metrics. Default: 0, no refresh of metrics. WARNING: This flag is not working since Kyverno 1.8.0 |
| metricsConfig.bucketBoundaries | list | `[0.005,0.01,0.025,0.05,0.1,0.25,0.5,1,2.5,5,10,15,20,25,30]` | Configures the bucket boundaries for all Histogram metrics, changing this configuration requires restart of the kyverno admission controller |
| metricsConfig.metricsExposure | map | `{"kyverno_admission_requests_total":{"disabledLabelDimensions":["resource_namespace"]},"kyverno_admission_review_duration_seconds":{"disabledLabelDimensions":["resource_namespace"]},"kyverno_cleanup_controller_deletedobjects_total":{"disabledLabelDimensions":["resource_namespace","policy_namespace"]},"kyverno_policy_execution_duration_seconds":{"disabledLabelDimensions":["resource_namespace","resource_request_operation"]},"kyverno_policy_results_total":{"disabledLabelDimensions":["resource_namespace","policy_namespace"]},"kyverno_policy_rule_info_total":{"disabledLabelDimensions":["resource_namespace","policy_namespace"]}}` | Configures the exposure of individual metrics, by default all metrics and all labels are exported, changing this configuration requires restart of the kyverno admission controller |

### Features

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| features.admissionReports.enabled | bool | `true` | Enables the feature |
| features.aggregateReports.enabled | bool | `true` | Enables the feature |
| features.policyReports.enabled | bool | `true` | Enables the feature |
| features.validatingAdmissionPolicyReports.enabled | bool | `false` | Enables the feature |
| features.reporting.validate | bool | `true` | Enables the feature |
| features.reporting.mutate | bool | `true` | Enables the feature |
| features.reporting.mutateExisting | bool | `true` | Enables the feature |
| features.reporting.imageVerify | bool | `true` | Enables the feature |
| features.reporting.generate | bool | `true` | Enables the feature |
| features.autoUpdateWebhooks.enabled | bool | `true` | Enables the feature |
| features.backgroundScan.enabled | bool | `true` | Enables the feature |
| features.backgroundScan.backgroundScanWorkers | int | `2` | Number of background scan workers |
| features.backgroundScan.backgroundScanInterval | string | `"1h"` | Background scan interval |
| features.backgroundScan.skipResourceFilters | bool | `true` | Skips resource filters in background scan |
| features.configMapCaching.enabled | bool | `true` | Enables the feature |
| features.controllerRuntimeMetrics.bindAddress | string | `":8080"` | Bind address for controller-runtime metrics (use "0" to disable it) |
| features.deferredLoading.enabled | bool | `true` | Enables the feature |
| features.dumpPayload.enabled | bool | `false` | Enables the feature |
| features.forceFailurePolicyIgnore.enabled | bool | `false` | Enables the feature |
| features.generateValidatingAdmissionPolicy.enabled | bool | `false` | Enables the feature |
| features.dumpPatches.enabled | bool | `false` | Enables the feature |
| features.globalContext.maxApiCallResponseLength | int | `2000000` | Maximum allowed response size from API Calls. A value of 0 bypasses checks (not recommended) |
| features.logging.format | string | `"text"` | Logging format |
| features.logging.verbosity | int | `2` | Logging verbosity |
| features.omitEvents.eventTypes | list | `["PolicyApplied","PolicySkipped"]` | Events which should not be emitted (possible values `PolicyViolation`, `PolicyApplied`, `PolicyError`, and `PolicySkipped`) |
| features.policyExceptions.enabled | bool | `false` | Enables the feature |
| features.policyExceptions.namespace | string | `""` | Restrict policy exceptions to a single namespace Set to "*" to allow exceptions in all namespaces |
| features.protectManagedResources.enabled | bool | `false` | Enables the feature |
| features.registryClient.allowInsecure | bool | `false` | Allow insecure registry |
| features.registryClient.credentialHelpers | list | `["default","google","amazon","azure","github"]` | Enable registry client helpers |
| features.ttlController.reconciliationInterval | string | `"1m"` | Reconciliation interval for the label based cleanup manager |
| features.tuf.enabled | bool | `false` | Enables the feature |
| features.tuf.root | string | `nil` | Path to Tuf root |
| features.tuf.rootRaw | string | `nil` | Raw Tuf root |
| features.tuf.mirror | string | `nil` | Tuf mirror |

### Admission controller

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| admissionController.autoscaling.enabled | bool | `false` | Enable horizontal pod autoscaling |
| admissionController.autoscaling.minReplicas | int | `1` | Minimum number of pods |
| admissionController.autoscaling.maxReplicas | int | `10` | Maximum number of pods |
| admissionController.autoscaling.targetCPUUtilizationPercentage | int | `80` | Target CPU utilization percentage |
| admissionController.autoscaling.behavior | object | `{}` | Configurable scaling behavior |
| admissionController.featuresOverride | object | `{"admissionReports":{"backPressureThreshold":1000}}` | Overrides features defined at the root level |
| admissionController.featuresOverride.admissionReports.backPressureThreshold | int | `1000` | Max number of admission reports allowed in flight until the admission controller stops creating new ones |
| admissionController.rbac.create | bool | `true` | Create RBAC resources |
| admissionController.rbac.createViewRoleBinding | bool | `true` | Create rolebinding to view role |
| admissionController.rbac.viewRoleName | string | `"view"` | The view role to use in the rolebinding |
| admissionController.rbac.serviceAccount.name | string | `nil` | The ServiceAccount name |
| admissionController.rbac.serviceAccount.annotations | object | `{}` | Annotations for the ServiceAccount |
| admissionController.rbac.coreClusterRole.extraResources | list | See [values.yaml](values.yaml) | Extra resource permissions to add in the core cluster role. This was introduced to avoid breaking change in the chart but should ideally be moved in `clusterRole.extraResources`. |
| admissionController.rbac.clusterRole.extraResources | list | `[]` | Extra resource permissions to add in the cluster role |
| admissionController.createSelfSignedCert | bool | `false` | Create self-signed certificates at deployment time. The certificates won't be automatically renewed if this is set to `true`. |
| admissionController.replicas | int | `nil` | Desired number of pods |
| admissionController.revisionHistoryLimit | int | `10` | The number of revisions to keep |
| admissionController.resyncPeriod | string | `"15m"` | Resync period for informers |
| admissionController.crdWatcher | bool | `false` | Enable/Disable custom resource watcher to invalidate cache |
| admissionController.podLabels | object | `{}` | Additional labels to add to each pod |
| admissionController.podAnnotations | object | `{}` | Additional annotations to add to each pod |
| admissionController.annotations | object | `{}` | Deployment annotations. |
| admissionController.updateStrategy | object | See [values.yaml](values.yaml) | Deployment update strategy. Ref: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy |
| admissionController.priorityClassName | string | `""` | Optional priority class |
| admissionController.apiPriorityAndFairness | bool | `false` | Change `apiPriorityAndFairness` to `true` if you want to insulate the API calls made by Kyverno admission controller activities. This will help ensure Kyverno stability in busy clusters. Ref: https://kubernetes.io/docs/concepts/cluster-administration/flow-control/ |
| admissionController.priorityLevelConfigurationSpec | object | See [values.yaml](values.yaml) | Priority level configuration. The block is directly forwarded into the priorityLevelConfiguration, so you can use whatever specification you want. ref: https://kubernetes.io/docs/concepts/cluster-administration/flow-control/#prioritylevelconfiguration |
| admissionController.hostNetwork | bool | `false` | Change `hostNetwork` to `true` when you want the pod to share its host's network namespace. Useful for situations like when you end up dealing with a custom CNI over Amazon EKS. Update the `dnsPolicy` accordingly as well to suit the host network mode. |
| admissionController.webhookServer | object | `{"port":9443}` | admissionController webhook server port in case you are using hostNetwork: true, you might want to change the port the webhookServer is listening to |
| admissionController.dnsPolicy | string | `"ClusterFirst"` | `dnsPolicy` determines the manner in which DNS resolution happens in the cluster. In case of `hostNetwork: true`, usually, the `dnsPolicy` is suitable to be `ClusterFirstWithHostNet`. For further reference: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy. |
| admissionController.dnsConfig | object | `{}` | `dnsConfig` allows to specify DNS configuration for the pod. For further reference: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config. |
| admissionController.startupProbe | object | See [values.yaml](values.yaml) | Startup probe. The block is directly forwarded into the deployment, so you can use whatever startupProbes configuration you want. ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/ |
| admissionController.livenessProbe | object | See [values.yaml](values.yaml) | Liveness probe. The block is directly forwarded into the deployment, so you can use whatever livenessProbe configuration you want. ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/ |
| admissionController.readinessProbe | object | See [values.yaml](values.yaml) | Readiness Probe. The block is directly forwarded into the deployment, so you can use whatever readinessProbe configuration you want. ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/ |
| admissionController.nodeSelector | object | `{}` | Node labels for pod assignment |
| admissionController.tolerations | list | `[]` | List of node taints to tolerate |
| admissionController.antiAffinity.enabled | bool | `true` | Pod antiAffinities toggle. Enabled by default but can be disabled if you want to schedule pods to the same node. |
| admissionController.podAntiAffinity | object | See [values.yaml](values.yaml) | Pod anti affinity constraints. |
| admissionController.podAffinity | object | `{}` | Pod affinity constraints. |
| admissionController.nodeAffinity | object | `{}` | Node affinity constraints. |
| admissionController.topologySpreadConstraints | list | `[]` | Topology spread constraints. |
| admissionController.podSecurityContext | object | `{}` | Security context for the pod |
| admissionController.podDisruptionBudget.enabled | bool | `false` | Enable PodDisruptionBudget. Will always be enabled if replicas > 1. This non-declarative behavior should ideally be avoided, but changing it now would be breaking. |
| admissionController.podDisruptionBudget.minAvailable | int | `1` | Configures the minimum available pods for disruptions. Cannot be used if `maxUnavailable` is set. |
| admissionController.podDisruptionBudget.maxUnavailable | string | `nil` | Configures the maximum unavailable pods for disruptions. Cannot be used if `minAvailable` is set. |
| admissionController.tufRootMountPath | string | `"/.sigstore"` | A writable volume to use for the TUF root initialization. |
| admissionController.sigstoreVolume | object | `{"emptyDir":{}}` | Volume to be mounted in pods for TUF/cosign work. |
| admissionController.caCertificates.data | string | `nil` | CA certificates to use with Kyverno deployments This value is expected to be one large string of CA certificates |
| admissionController.caCertificates.volume | object | `{}` | Volume to be mounted for CA certificates Not used when `.Values.admissionController.caCertificates.data` is defined |
| admissionController.imagePullSecrets | list | `[]` | Image pull secrets |
| admissionController.initContainer.image.registry | string | `nil` | Image registry |
| admissionController.initContainer.image.defaultRegistry | string | `"reg.kyverno.io"` |  |
| admissionController.initContainer.image.repository | string | `"kyverno/kyvernopre"` | Image repository |
| admissionController.initContainer.image.tag | string | `nil` | Image tag If missing, defaults to image.tag |
| admissionController.initContainer.image.pullPolicy | string | `nil` | Image pull policy If missing, defaults to image.pullPolicy |
| admissionController.initContainer.resources.limits | object | `{"cpu":"100m","memory":"256Mi"}` | Pod resource limits |
| admissionController.initContainer.resources.requests | object | `{"cpu":"10m","memory":"64Mi"}` | Pod resource requests |
| admissionController.initContainer.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}}` | Container security context |
| admissionController.initContainer.extraArgs | object | `{}` | Additional container args. |
| admissionController.initContainer.extraEnvVars | list | `[]` | Additional container environment variables. |
| admissionController.container.image.registry | string | `nil` | Image registry |
| admissionController.container.image.defaultRegistry | string | `"reg.kyverno.io"` |  |
| admissionController.container.image.repository | string | `"kyverno/kyverno"` | Image repository |
| admissionController.container.image.tag | string | `nil` | Image tag Defaults to appVersion in Chart.yaml if omitted |
| admissionController.container.image.pullPolicy | string | `"IfNotPresent"` | Image pull policy |
| admissionController.container.resources.limits | object | `{"memory":"384Mi"}` | Pod resource limits |
| admissionController.container.resources.requests | object | `{"cpu":"100m","memory":"128Mi"}` | Pod resource requests |
| admissionController.container.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}}` | Container security context |
| admissionController.container.extraArgs | object | `{}` | Additional container args. |
| admissionController.container.extraEnvVars | list | `[]` | Additional container environment variables. |
| admissionController.extraInitContainers | list | `[]` | Array of extra init containers |
| admissionController.extraContainers | list | `[]` | Array of extra containers to run alongside kyverno |
| admissionController.service.port | int | `443` | Service port. |
| admissionController.service.type | string | `"ClusterIP"` | Service type. |
| admissionController.service.nodePort | string | `nil` | Service node port. Only used if `type` is `NodePort`. |
| admissionController.service.annotations | object | `{}` | Service annotations. |
| admissionController.metricsService.create | bool | `true` | Create service. |
| admissionController.metricsService.port | int | `8000` | Service port. Kyverno's metrics server will be exposed at this port. |
| admissionController.metricsService.type | string | `"ClusterIP"` | Service type. |
| admissionController.metricsService.nodePort | string | `nil` | Service node port. Only used if `type` is `NodePort`. |
| admissionController.metricsService.annotations | object | `{}` | Service annotations. |
| admissionController.networkPolicy.enabled | bool | `false` | When true, use a NetworkPolicy to allow ingress to the webhook This is useful on clusters using Calico and/or native k8s network policies in a default-deny setup. |
| admissionController.networkPolicy.ingressFrom | list | `[]` | A list of valid from selectors according to https://kubernetes.io/docs/concepts/services-networking/network-policies. |
| admissionController.serviceMonitor.enabled | bool | `false` | Create a `ServiceMonitor` to collect Prometheus metrics. |
| admissionController.serviceMonitor.additionalLabels | object | `{}` | Additional labels |
| admissionController.serviceMonitor.namespace | string | `nil` | Override namespace |
| admissionController.serviceMonitor.interval | string | `"30s"` | Interval to scrape metrics |
| admissionController.serviceMonitor.scrapeTimeout | string | `"25s"` | Timeout if metrics can't be retrieved in given time interval |
| admissionController.serviceMonitor.secure | bool | `false` | Is TLS required for endpoint |
| admissionController.serviceMonitor.tlsConfig | object | `{}` | TLS Configuration for endpoint |
| admissionController.serviceMonitor.relabelings | list | `[]` | RelabelConfigs to apply to samples before scraping |
| admissionController.serviceMonitor.metricRelabelings | list | `[]` | MetricRelabelConfigs to apply to samples before ingestion. |
| admissionController.tracing.enabled | bool | `false` | Enable tracing |
| admissionController.tracing.address | string | `nil` | Traces receiver address |
| admissionController.tracing.port | string | `nil` | Traces receiver port |
| admissionController.tracing.creds | string | `""` | Traces receiver credentials |
| admissionController.metering.disabled | bool | `false` | Disable metrics export |
| admissionController.metering.config | string | `"prometheus"` | Otel configuration, can be `prometheus` or `grpc` |
| admissionController.metering.port | int | `8000` | Prometheus endpoint port |
| admissionController.metering.collector | string | `""` | Otel collector endpoint |
| admissionController.metering.creds | string | `""` | Otel collector credentials |
| admissionController.profiling.enabled | bool | `false` | Enable profiling |
| admissionController.profiling.port | int | `6060` | Profiling endpoint port |
| admissionController.profiling.serviceType | string | `"ClusterIP"` | Service type. |
| admissionController.profiling.nodePort | string | `nil` | Service node port. Only used if `type` is `NodePort`. |

### Background controller

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| backgroundController.featuresOverride | object | `{}` | Overrides features defined at the root level |
| backgroundController.enabled | bool | `true` | Enable background controller. |
| backgroundController.rbac.create | bool | `true` | Create RBAC resources |
| backgroundController.rbac.createViewRoleBinding | bool | `true` | Create rolebinding to view role |
| backgroundController.rbac.viewRoleName | string | `"view"` | The view role to use in the rolebinding |
| backgroundController.rbac.serviceAccount.name | string | `nil` | Service account name |
| backgroundController.rbac.serviceAccount.annotations | object | `{}` | Annotations for the ServiceAccount |
| backgroundController.rbac.coreClusterRole.extraResources | list | See [values.yaml](values.yaml) | Extra resource permissions to add in the core cluster role. This was introduced to avoid breaking change in the chart but should ideally be moved in `clusterRole.extraResources`. |
| backgroundController.rbac.clusterRole.extraResources | list | `[]` | Extra resource permissions to add in the cluster role |
| backgroundController.image.registry | string | `nil` | Image registry |
| backgroundController.image.defaultRegistry | string | `"reg.kyverno.io"` |  |
| backgroundController.image.repository | string | `"kyverno/background-controller"` | Image repository |
| backgroundController.image.tag | string | `nil` | Image tag Defaults to appVersion in Chart.yaml if omitted |
| backgroundController.image.pullPolicy | string | `"IfNotPresent"` | Image pull policy |
| backgroundController.imagePullSecrets | list | `[]` | Image pull secrets |
| backgroundController.replicas | int | `nil` | Desired number of pods |
| backgroundController.revisionHistoryLimit | int | `10` | The number of revisions to keep |
| backgroundController.resyncPeriod | string | `"15m"` | Resync period for informers |
| backgroundController.podLabels | object | `{}` | Additional labels to add to each pod |
| backgroundController.podAnnotations | object | `{}` | Additional annotations to add to each pod |
| backgroundController.annotations | object | `{}` | Deployment annotations. |
| backgroundController.updateStrategy | object | See [values.yaml](values.yaml) | Deployment update strategy. Ref: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy |
| backgroundController.priorityClassName | string | `""` | Optional priority class |
| backgroundController.hostNetwork | bool | `false` | Change `hostNetwork` to `true` when you want the pod to share its host's network namespace. Useful for situations like when you end up dealing with a custom CNI over Amazon EKS. Update the `dnsPolicy` accordingly as well to suit the host network mode. |
| backgroundController.dnsPolicy | string | `"ClusterFirst"` | `dnsPolicy` determines the manner in which DNS resolution happens in the cluster. In case of `hostNetwork: true`, usually, the `dnsPolicy` is suitable to be `ClusterFirstWithHostNet`. For further reference: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy. |
| backgroundController.dnsConfig | object | `{}` | `dnsConfig` allows to specify DNS configuration for the pod. For further reference: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config. |
| backgroundController.extraArgs | object | `{}` | Extra arguments passed to the container on the command line |
| backgroundController.extraEnvVars | list | `[]` | Additional container environment variables. |
| backgroundController.resources.limits | object | `{"memory":"128Mi"}` | Pod resource limits |
| backgroundController.resources.requests | object | `{"cpu":"100m","memory":"64Mi"}` | Pod resource requests |
| backgroundController.nodeSelector | object | `{}` | Node labels for pod assignment |
| backgroundController.tolerations | list | `[]` | List of node taints to tolerate |
| backgroundController.antiAffinity.enabled | bool | `true` | Pod antiAffinities toggle. Enabled by default but can be disabled if you want to schedule pods to the same node. |
| backgroundController.podAntiAffinity | object | See [values.yaml](values.yaml) | Pod anti affinity constraints. |
| backgroundController.podAffinity | object | `{}` | Pod affinity constraints. |
| backgroundController.nodeAffinity | object | `{}` | Node affinity constraints. |
| backgroundController.topologySpreadConstraints | list | `[]` | Topology spread constraints. |
| backgroundController.podSecurityContext | object | `{}` | Security context for the pod |
| backgroundController.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}}` | Security context for the containers |
| backgroundController.podDisruptionBudget.enabled | bool | `false` | Enable PodDisruptionBudget. Will always be enabled if replicas > 1. This non-declarative behavior should ideally be avoided, but changing it now would be breaking. |
| backgroundController.podDisruptionBudget.minAvailable | int | `1` | Configures the minimum available pods for disruptions. Cannot be used if `maxUnavailable` is set. |
| backgroundController.podDisruptionBudget.maxUnavailable | string | `nil` | Configures the maximum unavailable pods for disruptions. Cannot be used if `minAvailable` is set. |
| backgroundController.caCertificates.data | string | `nil` | CA certificates to use with Kyverno deployments This value is expected to be one large string of CA certificates |
| backgroundController.caCertificates.volume | object | `{}` | Volume to be mounted for CA certificates Not used when `.Values.backgroundController.caCertificates.data` is defined |
| backgroundController.metricsService.create | bool | `true` | Create service. |
| backgroundController.metricsService.port | int | `8000` | Service port. Metrics server will be exposed at this port. |
| backgroundController.metricsService.type | string | `"ClusterIP"` | Service type. |
| backgroundController.metricsService.nodePort | string | `nil` | Service node port. Only used if `metricsService.type` is `NodePort`. |
| backgroundController.metricsService.annotations | object | `{}` | Service annotations. |
| backgroundController.networkPolicy.enabled | bool | `false` | When true, use a NetworkPolicy to allow ingress to the webhook This is useful on clusters using Calico and/or native k8s network policies in a default-deny setup. |
| backgroundController.networkPolicy.ingressFrom | list | `[]` | A list of valid from selectors according to https://kubernetes.io/docs/concepts/services-networking/network-policies. |
| backgroundController.serviceMonitor.enabled | bool | `false` | Create a `ServiceMonitor` to collect Prometheus metrics. |
| backgroundController.serviceMonitor.additionalLabels | object | `{}` | Additional labels |
| backgroundController.serviceMonitor.namespace | string | `nil` | Override namespace |
| backgroundController.serviceMonitor.interval | string | `"30s"` | Interval to scrape metrics |
| backgroundController.serviceMonitor.scrapeTimeout | string | `"25s"` | Timeout if metrics can't be retrieved in given time interval |
| backgroundController.serviceMonitor.secure | bool | `false` | Is TLS required for endpoint |
| backgroundController.serviceMonitor.tlsConfig | object | `{}` | TLS Configuration for endpoint |
| backgroundController.serviceMonitor.relabelings | list | `[]` | RelabelConfigs to apply to samples before scraping |
| backgroundController.serviceMonitor.metricRelabelings | list | `[]` | MetricRelabelConfigs to apply to samples before ingestion. |
| backgroundController.tracing.enabled | bool | `false` | Enable tracing |
| backgroundController.tracing.address | string | `nil` | Traces receiver address |
| backgroundController.tracing.port | string | `nil` | Traces receiver port |
| backgroundController.tracing.creds | string | `""` | Traces receiver credentials |
| backgroundController.metering.disabled | bool | `false` | Disable metrics export |
| backgroundController.metering.config | string | `"prometheus"` | Otel configuration, can be `prometheus` or `grpc` |
| backgroundController.metering.port | int | `8000` | Prometheus endpoint port |
| backgroundController.metering.collector | string | `""` | Otel collector endpoint |
| backgroundController.metering.creds | string | `""` | Otel collector credentials |
| backgroundController.server | object | `{"port":9443}` | backgroundController server port in case you are using hostNetwork: true, you might want to change the port the backgroundController is listening to |
| backgroundController.profiling.enabled | bool | `false` | Enable profiling |
| backgroundController.profiling.port | int | `6060` | Profiling endpoint port |
| backgroundController.profiling.serviceType | string | `"ClusterIP"` | Service type. |
| backgroundController.profiling.nodePort | string | `nil` | Service node port. Only used if `type` is `NodePort`. |

### Cleanup controller

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| cleanupController.featuresOverride | object | `{}` | Overrides features defined at the root level |
| cleanupController.enabled | bool | `true` | Enable cleanup controller. |
| cleanupController.rbac.create | bool | `true` | Create RBAC resources |
| cleanupController.rbac.serviceAccount.name | string | `nil` | Service account name |
| cleanupController.rbac.serviceAccount.annotations | object | `{}` | Annotations for the ServiceAccount |
| cleanupController.rbac.clusterRole.extraResources | list | `[]` | Extra resource permissions to add in the cluster role |
| cleanupController.createSelfSignedCert | bool | `false` | Create self-signed certificates at deployment time. The certificates won't be automatically renewed if this is set to `true`. |
| cleanupController.image.registry | string | `nil` | Image registry |
| cleanupController.image.defaultRegistry | string | `"reg.kyverno.io"` |  |
| cleanupController.image.repository | string | `"kyverno/cleanup-controller"` | Image repository |
| cleanupController.image.tag | string | `nil` | Image tag Defaults to appVersion in Chart.yaml if omitted |
| cleanupController.image.pullPolicy | string | `"IfNotPresent"` | Image pull policy |
| cleanupController.imagePullSecrets | list | `[]` | Image pull secrets |
| cleanupController.replicas | int | `nil` | Desired number of pods |
| cleanupController.revisionHistoryLimit | int | `10` | The number of revisions to keep |
| cleanupController.resyncPeriod | string | `"15m"` | Resync period for informers |
| cleanupController.podLabels | object | `{}` | Additional labels to add to each pod |
| cleanupController.podAnnotations | object | `{}` | Additional annotations to add to each pod |
| cleanupController.annotations | object | `{}` | Deployment annotations. |
| cleanupController.updateStrategy | object | See [values.yaml](values.yaml) | Deployment update strategy. Ref: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy |
| cleanupController.priorityClassName | string | `""` | Optional priority class |
| cleanupController.hostNetwork | bool | `false` | Change `hostNetwork` to `true` when you want the pod to share its host's network namespace. Useful for situations like when you end up dealing with a custom CNI over Amazon EKS. Update the `dnsPolicy` accordingly as well to suit the host network mode. |
| cleanupController.server | object | `{"port":9443}` | cleanupController server port in case you are using hostNetwork: true, you might want to change the port the cleanupController is listening to |
| cleanupController.webhookServer | object | `{"port":9443}` | cleanupController webhook server port in case you are using hostNetwork: true, you might want to change the port the webhookServer is listening to |
| cleanupController.dnsPolicy | string | `"ClusterFirst"` | `dnsPolicy` determines the manner in which DNS resolution happens in the cluster. In case of `hostNetwork: true`, usually, the `dnsPolicy` is suitable to be `ClusterFirstWithHostNet`. For further reference: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy. |
| cleanupController.dnsConfig | object | `{}` | `dnsConfig` allows to specify DNS configuration for the pod. For further reference: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config. |
| cleanupController.extraArgs | object | `{}` | Extra arguments passed to the container on the command line |
| cleanupController.extraEnvVars | list | `[]` | Additional container environment variables. |
| cleanupController.resources.limits | object | `{"memory":"128Mi"}` | Pod resource limits |
| cleanupController.resources.requests | object | `{"cpu":"100m","memory":"64Mi"}` | Pod resource requests |
| cleanupController.startupProbe | object | See [values.yaml](values.yaml) | Startup probe. The block is directly forwarded into the deployment, so you can use whatever startupProbes configuration you want. ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/ |
| cleanupController.livenessProbe | object | See [values.yaml](values.yaml) | Liveness probe. The block is directly forwarded into the deployment, so you can use whatever livenessProbe configuration you want. ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/ |
| cleanupController.readinessProbe | object | See [values.yaml](values.yaml) | Readiness Probe. The block is directly forwarded into the deployment, so you can use whatever readinessProbe configuration you want. ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/ |
| cleanupController.nodeSelector | object | `{}` | Node labels for pod assignment |
| cleanupController.tolerations | list | `[]` | List of node taints to tolerate |
| cleanupController.antiAffinity.enabled | bool | `true` | Pod antiAffinities toggle. Enabled by default but can be disabled if you want to schedule pods to the same node. |
| cleanupController.podAntiAffinity | object | See [values.yaml](values.yaml) | Pod anti affinity constraints. |
| cleanupController.podAffinity | object | `{}` | Pod affinity constraints. |
| cleanupController.nodeAffinity | object | `{}` | Node affinity constraints. |
| cleanupController.topologySpreadConstraints | list | `[]` | Topology spread constraints. |
| cleanupController.podSecurityContext | object | `{}` | Security context for the pod |
| cleanupController.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}}` | Security context for the containers |
| cleanupController.podDisruptionBudget.enabled | bool | `false` | Enable PodDisruptionBudget. Will always be enabled if replicas > 1. This non-declarative behavior should ideally be avoided, but changing it now would be breaking. |
| cleanupController.podDisruptionBudget.minAvailable | int | `1` | Configures the minimum available pods for disruptions. Cannot be used if `maxUnavailable` is set. |
| cleanupController.podDisruptionBudget.maxUnavailable | string | `nil` | Configures the maximum unavailable pods for disruptions. Cannot be used if `minAvailable` is set. |
| cleanupController.service.port | int | `443` | Service port. |
| cleanupController.service.type | string | `"ClusterIP"` | Service type. |
| cleanupController.service.nodePort | string | `nil` | Service node port. Only used if `service.type` is `NodePort`. |
| cleanupController.service.annotations | object | `{}` | Service annotations. |
| cleanupController.metricsService.create | bool | `true` | Create service. |
| cleanupController.metricsService.port | int | `8000` | Service port. Metrics server will be exposed at this port. |
| cleanupController.metricsService.type | string | `"ClusterIP"` | Service type. |
| cleanupController.metricsService.nodePort | string | `nil` | Service node port. Only used if `metricsService.type` is `NodePort`. |
| cleanupController.metricsService.annotations | object | `{}` | Service annotations. |
| cleanupController.networkPolicy.enabled | bool | `false` | When true, use a NetworkPolicy to allow ingress to the webhook This is useful on clusters using Calico and/or native k8s network policies in a default-deny setup. |
| cleanupController.networkPolicy.ingressFrom | list | `[]` | A list of valid from selectors according to https://kubernetes.io/docs/concepts/services-networking/network-policies. |
| cleanupController.serviceMonitor.enabled | bool | `false` | Create a `ServiceMonitor` to collect Prometheus metrics. |
| cleanupController.serviceMonitor.additionalLabels | object | `{}` | Additional labels |
| cleanupController.serviceMonitor.namespace | string | `nil` | Override namespace |
| cleanupController.serviceMonitor.interval | string | `"30s"` | Interval to scrape metrics |
| cleanupController.serviceMonitor.scrapeTimeout | string | `"25s"` | Timeout if metrics can't be retrieved in given time interval |
| cleanupController.serviceMonitor.secure | bool | `false` | Is TLS required for endpoint |
| cleanupController.serviceMonitor.tlsConfig | object | `{}` | TLS Configuration for endpoint |
| cleanupController.serviceMonitor.relabelings | list | `[]` | RelabelConfigs to apply to samples before scraping |
| cleanupController.serviceMonitor.metricRelabelings | list | `[]` | MetricRelabelConfigs to apply to samples before ingestion. |
| cleanupController.tracing.enabled | bool | `false` | Enable tracing |
| cleanupController.tracing.address | string | `nil` | Traces receiver address |
| cleanupController.tracing.port | string | `nil` | Traces receiver port |
| cleanupController.tracing.creds | string | `""` | Traces receiver credentials |
| cleanupController.metering.disabled | bool | `false` | Disable metrics export |
| cleanupController.metering.config | string | `"prometheus"` | Otel configuration, can be `prometheus` or `grpc` |
| cleanupController.metering.port | int | `8000` | Prometheus endpoint port |
| cleanupController.metering.collector | string | `""` | Otel collector endpoint |
| cleanupController.metering.creds | string | `""` | Otel collector credentials |
| cleanupController.profiling.enabled | bool | `false` | Enable profiling |
| cleanupController.profiling.port | int | `6060` | Profiling endpoint port |
| cleanupController.profiling.serviceType | string | `"ClusterIP"` | Service type. |
| cleanupController.profiling.nodePort | string | `nil` | Service node port. Only used if `type` is `NodePort`. |

### Reports controller

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| reportsController.featuresOverride | object | `{}` | Overrides features defined at the root level |
| reportsController.enabled | bool | `true` | Enable reports controller. |
| reportsController.rbac.create | bool | `true` | Create RBAC resources |
| reportsController.rbac.createViewRoleBinding | bool | `true` | Create rolebinding to view role |
| reportsController.rbac.viewRoleName | string | `"view"` | The view role to use in the rolebinding |
| reportsController.rbac.serviceAccount.name | string | `nil` | Service account name |
| reportsController.rbac.serviceAccount.annotations | object | `{}` | Annotations for the ServiceAccount |
| reportsController.rbac.coreClusterRole.extraResources | list | See [values.yaml](values.yaml) | Extra resource permissions to add in the core cluster role. This was introduced to avoid breaking change in the chart but should ideally be moved in `clusterRole.extraResources`. |
| reportsController.rbac.clusterRole.extraResources | list | `[]` | Extra resource permissions to add in the cluster role |
| reportsController.image.registry | string | `nil` | Image registry |
| reportsController.image.defaultRegistry | string | `"reg.kyverno.io"` |  |
| reportsController.image.repository | string | `"kyverno/reports-controller"` | Image repository |
| reportsController.image.tag | string | `nil` | Image tag Defaults to appVersion in Chart.yaml if omitted |
| reportsController.image.pullPolicy | string | `"IfNotPresent"` | Image pull policy |
| reportsController.imagePullSecrets | list | `[]` | Image pull secrets |
| reportsController.replicas | int | `nil` | Desired number of pods |
| reportsController.revisionHistoryLimit | int | `10` | The number of revisions to keep |
| reportsController.resyncPeriod | string | `"15m"` | Resync period for informers |
| reportsController.podLabels | object | `{}` | Additional labels to add to each pod |
| reportsController.podAnnotations | object | `{}` | Additional annotations to add to each pod |
| reportsController.annotations | object | `{}` | Deployment annotations. |
| reportsController.updateStrategy | object | See [values.yaml](values.yaml) | Deployment update strategy. Ref: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy |
| reportsController.priorityClassName | string | `""` | Optional priority class |
| reportsController.apiPriorityAndFairness | bool | `false` | Change `apiPriorityAndFairness` to `true` if you want to insulate the API calls made by Kyverno reports controller activities. This will help ensure Kyverno reports stability in busy clusters. Ref: https://kubernetes.io/docs/concepts/cluster-administration/flow-control/ |
| reportsController.priorityLevelConfigurationSpec | object | See [values.yaml](values.yaml) | Priority level configuration. The block is directly forwarded into the priorityLevelConfiguration, so you can use whatever specification you want. ref: https://kubernetes.io/docs/concepts/cluster-administration/flow-control/#prioritylevelconfiguration |
| reportsController.hostNetwork | bool | `false` | Change `hostNetwork` to `true` when you want the pod to share its host's network namespace. Useful for situations like when you end up dealing with a custom CNI over Amazon EKS. Update the `dnsPolicy` accordingly as well to suit the host network mode. |
| reportsController.dnsPolicy | string | `"ClusterFirst"` | `dnsPolicy` determines the manner in which DNS resolution happens in the cluster. In case of `hostNetwork: true`, usually, the `dnsPolicy` is suitable to be `ClusterFirstWithHostNet`. For further reference: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy. |
| reportsController.dnsConfig | object | `{}` | `dnsConfig` allows to specify DNS configuration for the pod. For further reference: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config. |
| reportsController.extraArgs | object | `{}` | Extra arguments passed to the container on the command line |
| reportsController.extraEnvVars | list | `[]` | Additional container environment variables. |
| reportsController.resources.limits | object | `{"memory":"128Mi"}` | Pod resource limits |
| reportsController.resources.requests | object | `{"cpu":"100m","memory":"64Mi"}` | Pod resource requests |
| reportsController.nodeSelector | object | `{}` | Node labels for pod assignment |
| reportsController.tolerations | list | `[]` | List of node taints to tolerate |
| reportsController.antiAffinity.enabled | bool | `true` | Pod antiAffinities toggle. Enabled by default but can be disabled if you want to schedule pods to the same node. |
| reportsController.podAntiAffinity | object | See [values.yaml](values.yaml) | Pod anti affinity constraints. |
| reportsController.podAffinity | object | `{}` | Pod affinity constraints. |
| reportsController.nodeAffinity | object | `{}` | Node affinity constraints. |
| reportsController.topologySpreadConstraints | list | `[]` | Topology spread constraints. |
| reportsController.podSecurityContext | object | `{}` | Security context for the pod |
| reportsController.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}}` | Security context for the containers |
| reportsController.podDisruptionBudget.enabled | bool | `false` | Enable PodDisruptionBudget. Will always be enabled if replicas > 1. This non-declarative behavior should ideally be avoided, but changing it now would be breaking. |
| reportsController.podDisruptionBudget.minAvailable | int | `1` | Configures the minimum available pods for disruptions. Cannot be used if `maxUnavailable` is set. |
| reportsController.podDisruptionBudget.maxUnavailable | string | `nil` | Configures the maximum unavailable pods for disruptions. Cannot be used if `minAvailable` is set. |
| reportsController.tufRootMountPath | string | `"/.sigstore"` | A writable volume to use for the TUF root initialization. |
| reportsController.sigstoreVolume | object | `{"emptyDir":{}}` | Volume to be mounted in pods for TUF/cosign work. |
| reportsController.caCertificates.data | string | `nil` | CA certificates to use with Kyverno deployments This value is expected to be one large string of CA certificates |
| reportsController.caCertificates.volume | object | `{}` | Volume to be mounted for CA certificates Not used when `.Values.reportsController.caCertificates.data` is defined |
| reportsController.metricsService.create | bool | `true` | Create service. |
| reportsController.metricsService.port | int | `8000` | Service port. Metrics server will be exposed at this port. |
| reportsController.metricsService.type | string | `"ClusterIP"` | Service type. |
| reportsController.metricsService.nodePort | string | `nil` | Service node port. Only used if `type` is `NodePort`. |
| reportsController.metricsService.annotations | object | `{}` | Service annotations. |
| reportsController.networkPolicy.enabled | bool | `false` | When true, use a NetworkPolicy to allow ingress to the webhook This is useful on clusters using Calico and/or native k8s network policies in a default-deny setup. |
| reportsController.networkPolicy.ingressFrom | list | `[]` | A list of valid from selectors according to https://kubernetes.io/docs/concepts/services-networking/network-policies. |
| reportsController.serviceMonitor.enabled | bool | `false` | Create a `ServiceMonitor` to collect Prometheus metrics. |
| reportsController.serviceMonitor.additionalLabels | object | `{}` | Additional labels |
| reportsController.serviceMonitor.namespace | string | `nil` | Override namespace |
| reportsController.serviceMonitor.interval | string | `"30s"` | Interval to scrape metrics |
| reportsController.serviceMonitor.scrapeTimeout | string | `"25s"` | Timeout if metrics can't be retrieved in given time interval |
| reportsController.serviceMonitor.secure | bool | `false` | Is TLS required for endpoint |
| reportsController.serviceMonitor.tlsConfig | object | `{}` | TLS Configuration for endpoint |
| reportsController.serviceMonitor.relabelings | list | `[]` | RelabelConfigs to apply to samples before scraping |
| reportsController.serviceMonitor.metricRelabelings | list | `[]` | MetricRelabelConfigs to apply to samples before ingestion. |
| reportsController.tracing.enabled | bool | `false` | Enable tracing |
| reportsController.tracing.address | string | `nil` | Traces receiver address |
| reportsController.tracing.port | string | `nil` | Traces receiver port |
| reportsController.tracing.creds | string | `nil` | Traces receiver credentials |
| reportsController.metering.disabled | bool | `false` | Disable metrics export |
| reportsController.metering.config | string | `"prometheus"` | Otel configuration, can be `prometheus` or `grpc` |
| reportsController.metering.port | int | `8000` | Prometheus endpoint port |
| reportsController.metering.collector | string | `nil` | Otel collector endpoint |
| reportsController.metering.creds | string | `nil` | Otel collector credentials |
| reportsController.server | object | `{"port":9443}` | reportsController server port in case you are using hostNetwork: true, you might want to change the port the reportsController is listening to |
| reportsController.profiling.enabled | bool | `false` | Enable profiling |
| reportsController.profiling.port | int | `6060` | Profiling endpoint port |
| reportsController.profiling.serviceType | string | `"ClusterIP"` | Service type. |
| reportsController.profiling.nodePort | string | `nil` | Service node port. Only used if `type` is `NodePort`. |
| reportsController.sanityChecks | bool | `true` | Enable sanity check for reports CRDs |

### Grafana

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| grafana.enabled | bool | `false` | Enable grafana dashboard creation. |
| grafana.configMapName | string | `"{{ include \"kyverno.fullname\" . }}-grafana"` | Configmap name template. |
| grafana.namespace | string | `nil` | Namespace to create the grafana dashboard configmap. If not set, it will be created in the same namespace where the chart is deployed. |
| grafana.annotations | object | `{}` | Grafana dashboard configmap annotations. |
| grafana.labels | object | `{"grafana_dashboard":"1"}` | Grafana dashboard configmap labels |
| grafana.grafanaDashboard | object | `{"allowCrossNamespaceImport":true,"create":false,"folder":"kyverno","matchLabels":{"dashboards":"grafana"}}` | create GrafanaDashboard custom resource referencing to the configMap. according to https://grafana-operator.github.io/grafana-operator/docs/examples/dashboard_from_configmap/readme/ |

### Webhooks cleanup

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| webhooksCleanup.enabled | bool | `true` | Create a helm pre-delete hook to cleanup webhooks. |
| webhooksCleanup.autoDeleteWebhooks.enabled | bool | `false` | Allow webhooks controller to delete webhooks using finalizers |
| webhooksCleanup.image.registry | string | `nil` | Image registry |
| webhooksCleanup.image.repository | string | `"bitnami/kubectl"` | Image repository |
| webhooksCleanup.image.tag | string | `"1.32.3"` | Image tag Defaults to `latest` if omitted |
| webhooksCleanup.image.pullPolicy | string | `nil` | Image pull policy Defaults to image.pullPolicy if omitted |
| webhooksCleanup.imagePullSecrets | list | `[]` | Image pull secrets |
| webhooksCleanup.podSecurityContext | object | `{}` | Security context for the pod |
| webhooksCleanup.nodeSelector | object | `{}` | Node labels for pod assignment |
| webhooksCleanup.tolerations | list | `[]` | List of node taints to tolerate |
| webhooksCleanup.podAntiAffinity | object | `{}` | Pod anti affinity constraints. |
| webhooksCleanup.podAffinity | object | `{}` | Pod affinity constraints. |
| webhooksCleanup.podLabels | object | `{}` | Pod labels. |
| webhooksCleanup.podAnnotations | object | `{}` | Pod annotations. |
| webhooksCleanup.nodeAffinity | object | `{}` | Node affinity constraints. |
| webhooksCleanup.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsGroup":65534,"runAsNonRoot":true,"runAsUser":65534,"seccompProfile":{"type":"RuntimeDefault"}}` | Security context for the hook containers |
| webhooksCleanup.resources.limits | object | `{"cpu":"100m","memory":"256Mi"}` | Pod resource limits |
| webhooksCleanup.resources.requests | object | `{"cpu":"10m","memory":"64Mi"}` | Pod resource requests |

### Test

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| test.sleep | int | `20` | Sleep time before running test |
| test.image.registry | string | `nil` | Image registry |
| test.image.repository | string | `"busybox"` | Image repository |
| test.image.tag | string | `"1.35"` | Image tag Defaults to `latest` if omitted |
| test.image.pullPolicy | string | `nil` | Image pull policy Defaults to image.pullPolicy if omitted |
| test.imagePullSecrets | list | `[]` | Image pull secrets |
| test.resources.limits | object | `{"cpu":"100m","memory":"256Mi"}` | Pod resource limits |
| test.resources.requests | object | `{"cpu":"10m","memory":"64Mi"}` | Pod resource requests |
| test.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsGroup":65534,"runAsNonRoot":true,"runAsUser":65534,"seccompProfile":{"type":"RuntimeDefault"}}` | Security context for the test containers |

### Api version override

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| apiVersionOverride.podDisruptionBudget | string | `nil` | Override api version used to create `PodDisruptionBudget`` resources. When not specified the chart will check if `policy/v1/PodDisruptionBudget` is available to determine the api version automatically. |

### Other

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| global.image.registry | string | `nil` | Global value that allows to set a single image registry across all deployments. When set, it will override any values set under `.image.registry` across the chart. |
| global.imagePullSecrets | list | `[]` | Global list of Image pull secrets When set, it will override any values set under `imagePullSecrets` under different components across the chart. |
| global.resyncPeriod | string | `"15m"` | Resync period for informers |
| global.crdWatcher | bool | `false` | Enable/Disable custom resource watcher to invalidate cache |
| global.caCertificates.data | string | `nil` | Global CA certificates to use with Kyverno deployments This value is expected to be one large string of CA certificates Individual controller values will override this global value |
| global.caCertificates.volume | object | `{}` | Global value to set single volume to be mounted for CA certificates for all deployments. Not used when `.Values.global.caCertificates.data` is defined Individual  controller values will override this global value |
| global.extraEnvVars | list | `[]` | Additional container environment variables to apply to all containers and init containers |
| global.nodeSelector | object | `{}` | Global node labels for pod assignment. Non-global values will override the global value. |
| global.tolerations | list | `[]` | Global List of node taints to tolerate. Non-global values will override the global value. |
| nameOverride | string | `nil` | Override the name of the chart |
| fullnameOverride | string | `nil` | Override the expanded name of the chart |
| namespaceOverride | string | `nil` | Override the namespace the chart deploys to |
| upgrade.fromV2 | bool | `false` | Upgrading from v2 to v3 is not allowed by default, set this to true once changes have been reviewed. |
| rbac.roles.aggregate | object | `{"admin":true,"view":true}` | Aggregate ClusterRoles to Kubernetes default user-facing roles. For more information, see [User-facing roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles) |
| imagePullSecrets | object | `{}` | Image pull secrets for image verification policies, this will define the `--imagePullSecrets` argument |
| existingImagePullSecrets | list | `[]` | Existing Image pull secrets for image verification policies, this will define the `--imagePullSecrets` argument |
| customLabels | object | `{}` | Additional labels |
| policyReportsCleanup.enabled | bool | `true` | Create a helm post-upgrade hook to cleanup the old policy reports. |
| policyReportsCleanup.image.registry | string | `nil` | Image registry |
| policyReportsCleanup.image.repository | string | `"bitnami/kubectl"` | Image repository |
| policyReportsCleanup.image.tag | string | `"1.32.3"` | Image tag Defaults to `latest` if omitted |
| policyReportsCleanup.image.pullPolicy | string | `nil` | Image pull policy Defaults to image.pullPolicy if omitted |
| policyReportsCleanup.imagePullSecrets | list | `[]` | Image pull secrets |
| policyReportsCleanup.podSecurityContext | object | `{}` | Security context for the pod |
| policyReportsCleanup.nodeSelector | object | `{}` | Node labels for pod assignment |
| policyReportsCleanup.tolerations | list | `[]` | List of node taints to tolerate |
| policyReportsCleanup.podAntiAffinity | object | `{}` | Pod anti affinity constraints. |
| policyReportsCleanup.podAffinity | object | `{}` | Pod affinity constraints. |
| policyReportsCleanup.podLabels | object | `{}` | Pod labels. |
| policyReportsCleanup.podAnnotations | object | `{}` | Pod annotations. |
| policyReportsCleanup.nodeAffinity | object | `{}` | Node affinity constraints. |
| policyReportsCleanup.securityContext | object | `{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]},"privileged":false,"readOnlyRootFilesystem":true,"runAsGroup":65534,"runAsNonRoot":true,"runAsUser":65534,"seccompProfile":{"type":"RuntimeDefault"}}` | Security context for the hook containers |
| policyReportsCleanup.resources.limits | object | `{"cpu":"100m","memory":"256Mi"}` | Pod resource limits |
| policyReportsCleanup.resources.requests | object | `{"cpu":"10m","memory":"64Mi"}` | Pod resource requests |

## TLS Configuration

If `admissionController.createSelfSignedCert` is `true`, Helm will take care of the steps of creating an external self-signed certificate described in option 2 of the [installation documentation](https://kyverno.io/docs/installation/#option-2-use-your-own-ca-signed-certificate)

If `admissionController.createSelfSignedCert` is `false`, Kyverno will generate a self-signed CA and a certificate, or you can provide your own TLS CA and signed-key pair and create the secret yourself as described in the [documentation](https://kyverno.io/docs/installation/#customize-the-installation-of-kyverno).

## Default resource filters

[Kyverno resource filters](https://kyverno.io/docs/installation/#resource-filters) are a used to exclude resources from the Kyverno engine rules processing.

This chart comes with default resource filters that apply exclusions on a couple of namespaces and resource kinds:
- all resources in `kube-system`, `kube-public` and `kube-node-lease` namespaces
- all resources in all namespaces for the following resource kinds:
  - `Event`
  - `Node`
  - `APIService`
  - `TokenReview`
  - `SubjectAccessReview`
  - `SelfSubjectAccessReview`
  - `Binding`
  - `ReplicaSet`
  - `AdmissionReport`
  - `ClusterAdmissionReport`
  - `BackgroundScanReport`
  - `ClusterBackgroundScanReport`
- all resources created by this chart itself

Those default exclusions are there to prevent disruptions as much as possible.
Under the hood, Kyverno installs an admission controller for critical cluster resources.
A cluster can become unresponsive if Kyverno is not up and running, ultimately preventing pods to be scheduled in the cluster.

You can however override the default resource filters by setting the `config.resourceFilters` stanza.
It contains an array of string templates that are passed through the `tpl` Helm function and joined together to produce the final `resourceFilters` written in the Kyverno config map.

Please consult the [values.yaml](values.yaml) file before overriding `config.resourceFilters` and use the apropriate templates to build your desired exclusions list.

Add entries to `config.resourceFiltersExclude` that you wish to omit from `config.resourceFilters`.

Add entries to `config.resourceFiltersInclude` that you with to add to `config.resourceFilters`.

## High availability

Running a highly-available Kyverno installation is crucial in a production environment.

In order to run Kyverno in high availability mode, you should set `replicas` to `3` or more for desired components.
You should also pay attention to anti affinity rules, spreading pods across nodes and availability zones.

Please see https://kyverno.io/docs/installation/#security-vs-operability for more informations.

## Source Code

* <https://github.com/kyverno/kyverno>

## Requirements

Kubernetes: `>=1.25.0-0`

| Repository | Name | Version |
|------------|------|---------|
|  | crds | 3.4.4 |
|  | grafana | 3.4.4 |

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| Nirmata |  | <https://kyverno.io/> |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.11.0](https://github.com/norwoodj/helm-docs/releases/v1.11.0)
