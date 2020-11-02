This repository can be used as a template for Kubernetes deployment information.
This deployment template provides the base files to deploy on Kubernetes through Kustomize.

# Files
A deployment repository should contain all relevant Kubernetes deployment information, either through plain yaml manifests, Kustomize or Helm. 
It should also contain all cloud specific configuration, preferably through the use of ConfigMaps.

# Sensitive information
Sensitive data should be handled through Sealed Secrets. Follow the steps provided in https://github.com/bitnami-labs/sealed-secrets#usage to generate a plain Kubernetes secret locally. By using the foo-secret.json format and this template, the plain secret will automatically be ignored by git. 

Convert the secret to a Sealed Secret for each environment (generally staging and production), using the naming scheme foo-sealedsecret.json. 

> Take care to keep track of the sensitive data out-of-band, as you can't retrieve the original values from the sealed secret without Kubernetes access! In case anything goes wrong with the Kubernetes cluster or the Sealed Secrets controller, your data might be lost!

# Kustomize layout
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
>  When using this option, make sure to update your references to the base folder, e.g. in `kustomization.yaml`

2. prefixed
```
overlays
|-- staging-setup1
|-- staging-setup2
|-- production
```
