# Fleet

## High level definitions

- Fleet Manager: centralized component that orchestrates deployments of k8s assets from hit repos. One per cluster, be it a single or multiclister.
- Fleet controller: Runs on/is part of fleet manager, performs GitOps orchestration.
- Fleet agent: Controllers on downstream for communicating with fleet manager upstream.
- GitRepo: A repository that is watched by fleet
- Bundle: Processed GitRepos are turned into one or more Bundles depending on configuration in the GitRepo. Bundles are one or more of: k8s manifest, kustomize config, or helm charts. The contents are rendered into a helm chart by the agent and installed into the downstream cluster as a helm release.
- BundleDeployment: State of a Bundle on a specific cluster with its cluster customizations. The agent is only aware of BundleDeployment resources that are created for the cluster the agent is managing.

## Bundle Lifecycle

1. User creates GitRepo
1. gitjob-controller syncs changes from the GitRepo and detects changes from the polling or webhook event. gitjob-controller will create a job that clones the git repository, reads contents, and creates a bundle for every commit change.
1. The fleet-controller syncs changes from the bundle. According to the targets, the fleet-controller will create BundleDeployments.
1. fleet-agent will pull the BundleDeployment from the controlplane. The agent delpoys bundle manifests as a helm chart from the BundleDeployment.
1. fleet-agent will continue to monitor the application bundle and report back status in this order: bundledeployment > bundle > gitrepo > cluster

https://fleet.rancher.io/assets/images/FleetBundleStages-266005b85e14d0b48d2e1067b8641f83.svg

## Git repo contents

Bundles are created from either found fleet.yaml or specifying paths manually during creation fo the GitRepo. Bundle lifecycles are tracked between releases by the helm releaseName field added to each bundle. If the releaseName is not specified within fleet.yaml it is generated from GitRepo.name + path. Long names are truncated and a -<hash> prefix is added.

The git repository has no explicitly required structure. It is important to realize the scanned resources will be saved as a resource in Kubernetes so you want to make sure the directories you are scanning in git do not contain arbitrarily large resources. Right now there is a limitation that the resources deployed must gzip to less than 1MB.

- Chart.yaml: / relative to path or custom path from fleet.yaml The resources will be deployed as a Helm chart. Refer to the fleet.yaml for more options.
- kustomization.yaml: / relative to path or custom path from fleet.yaml The resources will be deployed using Kustomize. Refer to the fleet.yaml for more options.
- fleet.yaml: Any subpath If any fleet.yaml is found a new bundle will be defined. This allows mixing charts, kustomize, and raw YAML in the same repo
- *.yaml  Any subpath If a Chart.yaml or kustomization.yaml is not found then any .yaml or .yml file will be assumed to be a Kubernetes resource and will be deployed.
- overlays/{name} / relative to path When deploying using raw YAML (not Kustomize or Helm) overlays is a special directory for customizations.

Fleet supports file and directory exclusion by means of .fleetignore files, in a similar fashion to how .gitignore files behave in git repositories:

- Glob syntax is used to match files or directories, using Golang's filepath.Match
- Empty lines are skipped, and can therefore be used to improve readability
- Characters like white spaces and # can be escaped with a backslash
- Trailing spaces are ignored, unless escaped
- Comments, ie lines starting with unescaped #, are skipped
- A given line can match a file or a directory, even if no separator is provided: eg. subdir/* and subdir are both valid .fleetignore lines, and subdir matches both files and directories called subdir
- A match may be found for a file or directory at any level below the directory where a .fleetignore lives, ie foo.yaml will match ./foo.yaml as well as ./path/to/foo.yaml
- Multiple .fleetignore files are supported. For instance, in the following directory structure, only root/something.yaml, bar/something2.yaml and foo/something.yaml will end up in a bundle:

This currently comes with a few limitations, the following not being supported:

- Double asterisks (**)
- Explicit inclusions with !

## fleet.yaml

Optional, changes how resources are deployed and customized. It should always be at the root or path root specified. If a subdir has a fleet.yaml, a new bundle is defined for that differently than the parent.

Helm chart dependencies: It is up to the user to fulfill the dependency list for the Helm charts. As such, you must manually run helm dependencies update $chart OR run helm dependencies build $chart prior to install. See the Fleet docs in Rancher for more information.

Available fields are documented here: https://fleet.rancher.io/ref-fleet-yaml

Using secrets for repositories: https://fleet.rancher.io/gitrepo-add#using-private-helm-repositories

## values.yaml (helm)

The most recently applied changes to values.yaml will override any previously existing values. If changes are applied from multiple sources at the same time, the value will update in this order: helm.values -> helm.valuesFiles -> helm.valuesFrom. That means valuesFrom will take precedence.

### using valuesFrom

If you want to use the helm.valuesFrom key, you specify your API resources like so:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-values
  namespace: default
data:
  values.yaml: |-
    replication: true
    replicas: 2
    serviceType: NodePort
---
apiVersion: v1
kind: Secret
metadata:
  name: secret-values
  namespace: default
stringData:
  values.yaml: |-
    replication: true
    replicas: 3
    serviceType: NodePort
```

You can then use this configmap or secret like so in helm:

```
helm:
  chart: simple-chart
  valuesFrom:
    - secretKeyRef:
        name: secret-values
        namespace: default
        key: values.yaml
    - configMapKeyRef:
        name: configmap-values
        namespace: default
        key: values.yaml
  values:
    replicas: "4"
```

## Per-cluster customization

