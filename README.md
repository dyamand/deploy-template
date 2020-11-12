This repository can be used as a template for Kubernetes deployment information.
This deployment template provides the base files to deploy on Kubernetes through Kustomize.

# Files
A deployment repository should contain all relevant Kubernetes deployment information, either through plain yaml manifests, Kustomize or Helm. 
It should also contain all cloud specific configuration, preferably through the use of ConfigMaps.

# Sensitive information
Sensitive data should be handled through Sealed Secrets. Follow the steps provided in https://github.com/bitnami-labs/sealed-secrets#usage to generate a plain Kubernetes secret locally. By using the foo-secret.json format and this template, the plain secret will automatically be ignored by git. 

Convert the secret to a Sealed Secret for each environment (generally staging and production), using the naming scheme foo-sealedsecret.json. 
You might have to remove the empty `status` field from the generated manifests, as an empty field in the manifest combined with an 'omitempty' trait in the resource definition might result in the GitOps engine seeing the object as out-of-sync.

> Take care to keep track of the sensitive data out-of-band, as you can't retrieve the original values from the sealed secret without Kubernetes access! In case anything goes wrong with the Kubernetes cluster or the Sealed Secrets controller, your data might be lost!

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

## Labels
When using 'commonLabels', take into account that the labels are applied per kustomization file. If you have a base file that defines a deployment, the common labels from that base kustomization file will be applied to the deployment. If an overlay adds a new service, the common labels of that overlay kustomization file will be added to the service. The service won't have the common labels of the base kustomization file, nor will the deployment have the common labels of the overlay.

## ConfigMaps
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

By default, kustomize generators add a hash to the name of their generated resource, to prevent overwriting existing resources. We generally don't use this functionality, it can be disabled by adding 
```
generatorOptions:
  disableNameSuffixHash: true
```
If you do want to use this functionality, know that kustomize will automatically replace any reference to the generated resource within the defined manifests with the hash-suffixed name.
