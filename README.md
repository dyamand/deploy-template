This repository can be used as a template for Kubernetes deployment information.
This deployment template provides the base files to deploy on Kubernetes through Kustomize.

# Files
A deployment repository should contain all relevant Kubernetes deployment information, either through plain yaml manifests, Kustomize or Helm. 
It should also contain all cloud specific configuration, preferably through the use of ConfigMaps.

# Sensitive information
Sensitive data should be handled through Sealed Secrets. Follow the steps provided in https://github.com/bitnami-labs/sealed-secrets#usage to generate a plain Kubernetes secret locally. By using the foo-secret.json format and this template, the plain secret will automatically be ignored by git. 

Convert the secret to a Sealed Secret for each environment (generally staging and production), using the naming scheme foo-sealedsecret.json. 
You might have to remove the empty `status` field from the generated manifests, as an empty field in the manifest combined with an 'omitempty' trait in the resource definition might result in the GitOps engine seeing the object as out-of-sync.

---
Keep a copy of sensitive data out-of-band! In case anything goes wrong with the Kubernetes cluster or the Sealed Secrets controller, your data might be lost!
---

> Usage of sealed secrets requires you to have a valid kube config file locally. You might not have the correct access rights to create sealed secrets. If this is the case, please contact the cluster administrators.

# Kustomize 
## Layout
Use the `base` folder to define everything that is shared between environments.
Overlays can be used to specify environment-specific configuration. 
In some cases, multiple deployments per environment are desired. Simply create a new overlay and give it an appropriate name. 
Make sure that it is clear what environment your overlay is meant for.
A few examples of how different overlays can be handled:
1. nested:
```
overlays
|-- staging
|   |-- setup1
|   |-- setup2
|-- production
```
>  When using this option, make sure to update your relative references to the base folder, e.g. in `overlays/staging/setup1/kustomization.yaml`

2. prefixed
```
overlays
|-- staging-setup1
|-- staging-setup2
|-- production
```

## Patches
When using overlays, you might want to overwrite fields set by your base directory. This can be done through the use of patch files. Kustomize supports 3 types of patches:

* patchesStrategicMerge: A list of patch files where each file is parsed as a Strategic Merge Patch.
* patchesJSON6902: A list of patches and associated targetes, where each file is parsed as a JSON Patch and can only be applied to one target resource.
* patches: A list of patches and their associated targets. The patch can be applied to multiple objects. It auto detects whether the patch is a Strategic Merge Patch or JSON Patch.
> taken from https://github.com/kubernetes-sigs/kustomize/blob/master/examples/inlinePatch.md, see link for inline examples for each patch type

In our case, we rely mostly on patchesStrategicMerge and a separate patch file. By using these patches, our CICD pipeline can automatically update image tags according to their environment. See our deployment info on confluence for more information on how the source repo can be configured to trigger automatic updates in the deployment repo.

## Labels
When using 'commonLabels', take into account that the labels are applied per kustomization file. If you have a base file that defines a deployment, the common labels from that base kustomization file will be applied to the deployment. If an overlay adds a new service, the common labels of that overlay kustomization file will be added to the service. The service won't have the common labels of the base kustomization file, nor will the deployment have the common labels of the overlay.

## ConfigMaps
### Generator types
While one can add plain ConfigMap manifests to the kustomization resources, kustomize provides ConfigMapGenerators to automatically generate ConfigMaps from other files. There are a couple of different options to generate ConfigMaps from files, they are:
* literals: literal key=value pairs
* envs: each line in the given files should be in the form of key=value
* files: the content of each file is used as value, by default the name is used as key, but this can be overwritten by using the syntax `keyname=path/to/file`

To use a ConfigMapGenerator, add it to the kustomization.yaml. Examples:
```
configMapGenerator:
- name: the-map
  literals:
  - altGreeting=Good Morning!
  - enableRisky="false"
- name: another-map
  files:
  - foo.bar
  - thisIsACustomKey=foo2.bar
  envs:
  - foo3.env
```

### Generator options
By default, kustomize generators add a hash to the name of their generated resource, to prevent overwriting existing resources. We generally don't use this functionality, it can be disabled by adding 
```
generatorOptions:
  disableNameSuffixHash: true
```
If you do want to use this functionality, know that kustomize will automatically replace any reference to the generated resource within the defined manifests with the hash-suffixed name.

### Merge behaviour
It is possible to set what action to take upon generating a new ConfigMap or Secret, the options are:
* create: attempt to create a new object, throw an error if it already exists
* merge: merge with existing object, throw an error if there is no existing object
* replace: overwrite an existing object, throw an error if there is no existing object

To set what behavior to use, simply add the corresponding type to your generator:
```
configMapGenerator:
- name: config-map
  behavior: merge
  [...]
```

Our general modus operandi is to define any configuration shared between environments in one or more configuration files in the base directory. Any environmnet specific configuration can then be configured in specific overlays. By defining ConfigMapGenerators in both the base kustomization file as well as in the overlay kustomization file and setting the merge behaviour of the overlay ConfigMapGenerators to merge, one can combine base and overlay configuration in the same ConfigMap(s).

# Using data from other repositories
In some specific cases, one might want to use configuration parameters defined in another repository, e.g. shared database connection information.
In such cases, it is advised to disable any pre- and suffix generation, as well as hash generation, for those shared resources, as kustomize will only update named references in the manifests defined through its kustomization file.

If no custom affixes are added to the resource, one can simply refer to an existing ConfigMap or Secret by name. 
