---
author: "Matic Rupnik"
authorLink: matic-rupnik
date: 2024-07-21    
title: Managing SOPS secrets for Kubernetes deployments ft. ArgoCD
tags: [
  "ArgoCD",
  "Kubernetes",
  "sops",
  "Helm",
  "Docker",
]
---

I have come across the issue of having to deal with scaling and automating a Kubernetes helmfile driven deployment. The main need was to have the contents of Kubernetes secrets stored in separate _values.yaml_ files to be encrypted using sops as until now and to have this Helm chart be compatible with ArgoCD out of the box without configuring extra plugins as to align the application to any GitOps or raw Helm driven environments at different customers. I achieved this with a custom decryption container that can run at different times in the deployment process but I will focus on the single Deployment and InitContainer method for the sake of this demonstration. There is no one method to achieve this but I hope this article provides some ideas to your use cases.
You can find the resources at:
- [Git repository](https://github.com/mrupnikm/Captain-Olm)
- [custom image](https://hub.docker.com/repository/docker/mrupnikm/olm-chart-sops-decryption/)
- [Helm chart](../../charts/olm-chart/)
            
> Note that the example chart is a oversimplified version of my use case scenario. I will point out where you can make changes.

# Component overview
Captain Olm chart was started on the base of `helm create` utility that creates a boilerplate with the deployment strategy in mind so some of the values keys should be familiar to you.

## In a nut shell it works like this (tldr)
This chart expects to have a secret yaml file encrypted with SOPS age (or PGP) without a passphrase and a copy of the respective private key in a secret in the same namespace as the deployment under the name of `<chartname>-age-keys` (or  `<chartname>-pgp-keys`) and value of `age-key.txt` (or `pgp-private-key.asc`). The encrypted secret file is set with `helm <...> --set-file extraSecretFile=secret-value.enc.yaml` that will get written into `<chartname>-encrypted-secret` and translated in flight in the deployments InitContainer(or Helm life-cycled managed Job) and writes the contents to `<chartname>-secret` using kubectl (hence the need for a service account in the chart). This scenario works with ArgoCD (that implement the feature supporting the `--set-file` command) out of the box without the addition of plugins.

## The architecture and configuration
The setup uses some extra objects to help with the creation of the new secret in the namespace such as service, role and rolebinding. As for the whole chart, this essentially expects configmap data, an encrypted yaml file for the secret transformation, a initContainer image and the encrypted secret type

```yaml
# the value of this key gets transalted into the configmap 
configMap:  |
  kube-props.kubes[0].name=example
  
# this value gets populated with the encrypted yaml file on using --set-file flag so it should always be empty 
extraSecretFile: ""

initContainer:  
  image: x  
  # for setting namespace or other configuration for the applying of the secret
  k8s_args: "-n yournamespace"
  
encrypted_secret:  
  # if using age than the value of the key can be anything not nullable 
  age: "x"
  # if using pgp than the value should be the public key indentifier for the keychain
  #pgp: "dsgff4sfr534645..."  

```

It is vital that you also use the `nameOverride` key that will rename the chart and most of the kubernetes objects in the setup. This also effects your ingress configuration, so if you used "mywebapp" as the name and "lab.com" as the ingress domain it will be appended together as "mywebapp.lab.com".

## The magic container
The beating heart of this chart is the docker image with a script. Essentially the docker image includes SOPS and kubectl binaries to the base image of your choice and adds the `decrypt-sops.sh` script. The script takes in the parameters of MODE, PGP-KEY, NEW-SECRET-NAME, K8S-ARGS and HELM_NAME(you can find the usage in `_decryptionInitContainer.tpl`).
As explained above the main purpose is to read the `<chart-name>-encrypted-secret` and set the new `<chart-name>-secret` that is mounted in the deployments containers. To change the end secrets to a different layout tweak the `handle_secret_creation` function.

As mentioned in the example helm-initContainer I use the initContainer strategy to apply these changes but the same can be achieved with Job objects that runs before the main deployment/s are started with the help of Helm hooks.

Now, let me explain an example scenario for a better understanding.

# Scenario 1: Copied chart directory
Let's say that you want to package a Helm chart that deploys a simple web application with configurable configmap and encrypted secrets in form of a separate values file that will be easily editable using SOPS plugins like the idea plugin. I will be focusing on using age encryption in this example but I have made this chart to be compatible with PGP passphrase-less encryption as well. I will also focus on having this charts configuration in the application git repository.

## Prerequisites
- SOPS utility installed (OPTIONAL: idea SOPS plugin)
- Docker, Helm and kubectl utility installed
- configured ArgoCD and its utility installed
- helm-initContainer directory cloned in your project repository
- make the custom image accessible to your cluster (there are configuration keys set values)

_custom-values.yaml:_
```yaml
configMap: |
  kube-props.kubes[0].name=example
  
extraSecretFile: ""
```

_secret-values.yaml:_
```yaml
data: "some secret you want to protect"
```

## Steps to deployment

1.  Encrypt the data `secret-values.yaml` file using age SOPS
    1. create age key `age-keygen -o age-key.txt` (make sure not to commit this to GIT)
    2. copy the private key to the `.sops.yaml` file
    3. `sops -e argocd/helm/secret.yaml > argocd/helm/secret.enc.yaml`
2.  Configure the a custom-values.yaml as needed
    1. make note of the `nameOverride` as this will be the name of your chart and deployment later
    2. in case of age encryption set this key `encrypted_secret.age: ""`
3. Set a secret containing the age private key as its contents in the deployment namespace(found in `age-key.txt`)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: age-keys
  namespace: webappnamespace
data:
  age-key.txt: >- ...........
```
4.  Configure the ArgoCD file
    1. set the namespace of the deployment
    2. configure and git repository to be accessible to ArgoCD
    3. change the name of your encrypted secret file under `fileParameters`
5.  Deploy the application using the CLI with `argocd app create -f argo.yaml` or over the Web interface pasting the configuration yaml.

The application should be ready and deployed.
# Scenario 2: Published chart
Lets say you have a published **captain-olm** chart and you would like to use it in your project. For that you can clone the contents of examples/umbrella-example to the root of your project.
there you will find a few files:
- `argo.yaml` for ArgoCD deployment
- `.sops.yaml` for defining your encryption
- helm directory with:
    - `Chart.yaml` where you list the **captain-olm** as a dependency
    - `values.yaml` that same as in Scenario 1 but indentation to comply with umbrella chart passing of values
    - `secrets-values.yaml` that you encrypt using SOPS.

_Chart.yaml:_
```yaml
apiVersion: v2  
name: captain-olm  
description: An umbrella chart for managing custom values for dependent charts  
version: 0.1.0  
dependencies:  
  - name: captain-olm  
    version: 0.1.3  
    repository: "https://repo.example.com"
```

_values.yaml:_
```yaml
captain-olm:  
  nameOverride: "webapp"
...
```

In general the steps to deploy are the same as in the above scenario. After configuration you apply the ArgoCD manifest and the application should be up

# Notes
If you are cleaning up the project the make sure that you delete the decryption secret as it is the only file not tracked by ArgoCD.

# Closing thoughts

The approach I have orchestrated is by no means the best option for handling secrets in deployments, but it did fit the use case I had quite well. You could choose to use make the decryption on the application level and not leave it to the orchestration layer or not handle it with SOPS encryption altogether and use HCP Vault or alike. I hope that my example gives you an easy boilerplate or just an idea on how to handle your use cases. Happy Helming!