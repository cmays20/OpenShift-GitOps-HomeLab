# OpenShift HomeLab Setup using GitOps

This project is the GitOps Repository for my HomeLab.

## Prerequisites 

- OpenShift cluster installed
- OpenShift GitOps Operator installed in cluster

## Making the Masters Schedulable

```shell
oc patch scheduler/cluster -p '{"spec":{"mastersSchedulable":true}}' --type=merge
```

## Setting up ArgoCD

### 1. Change the label that is used to identify ArgoCD owned objects

The default label used by ArgoCD is: `app.kubernetes.io/instance: some-application`.
This label is sometimes used by other projects as well. This creates a conflict and can have negative side effects.
Therefore, it is best to change it to something custom.  Reference: [ArgoCD Resource Tracking](https://argo-cd.readthedocs.io/en/latest/user-guide/resource_tracking/).

Add the following to the ArgoCD object that is created when installing the cluster:
```yaml
  extraConfig:
    application.instanceLabelKey: argocd.argoproj.io/instance
```

### 2. Install KSOPS

`KSOPS`, or kustomize-SOPS, is a [kustomize](https://github.com/kubernetes-sigs/kustomize/) 
[KRM exec plugin](https://kubectl.docs.kubernetes.io/guides/extending_kustomize/exec_krm_functions/) 
for SOPS encrypted resources. `KSOPS` can be used to decrypt any Kubernetes resource, 
but is most commonly used to decrypt encrypted Kubernetes Secrets and ConfigMaps. 
As a kustomize plugin, `KSOPS` allows you to manage, build, 
and apply encrypted manifests the same way you manage the rest of your Kubernetes manifests.
Reference: [KSOPS](https://github.com/viaduct-ai/kustomize-sops)

#### Install Kustomize

You can use the following to install [Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/):
```shell
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```

#### Install SOPS

The instructions for this can be found [here](https://github.com/getsops/sops/releases)

#### Install KSOPS



#### Install Age Tool

You can find further instructions [here](https://github.com/FiloSottile/age#installation)
```shell
wget -O age-tool.tar.gz https://dl.filippo.io/age/latest?for=linux/amd64
```
You just need to untar the file and make sure the resulting executables are in your path.

#### Configure SOPS

```shell
age-keygen -o age.agekey
```

Copy the public key above into the below command:
```shell
cat <<EOF > .sops.yaml
creation_rules:
  - path_regex: apps/.*\.sops\.ya?ml
    encrypted_regex: "^(data|stringData)$"
    age: <YOUR PUBLIC KEY HERE>
EOF
```

#### Configure ArgoCD

First, create a secret from the `age.agekey` file that was created in the previous section.
```shell
cat age.agekey | oc create secret generic sops-age --namespace=openshift-gitops \
--from-file=key.txt=/dev/stdin
```

Then, we need to update the ArgoCD cluster yaml again, now we need to add the following:
```yaml
  kustomizeBuildOptions: --enable-alpha-plugins --enable-exec
  repo:
    env:
    - name: XDG_CONFIG_HOME
      value: /.config
    - name: SOPS_AGE_KEY_FILE
      value: /.config/sops/age/key.txt
    initContainers:
    - args:
      - echo "Installing KSOPS..."; mv ksops /custom-tools/; mv kustomize /custom-tools/;
        echo "Done.";
      command:
      - /bin/sh
      - -c
      image: viaductoss/ksops:v4.3.2
      name: install-ksops
      volumeMounts:
      - mountPath: /custom-tools
        name: custom-tools
    volumeMounts:
    - mountPath: /usr/local/bin/kustomize
      name: custom-tools
      subPath: kustomize
    - mountPath: /usr/local/bin/ksops
      name: custom-tools
      subPath: ksops
    - mountPath: /.config/sops/age/key.txt
      name: sops-age
      subPath: key.txt
    volumes:
    - emptyDir: {}
      name: custom-tools
    - name: sops-age
      secret: sops-age
```

#### ArgoCD access to private repository (Optional: Depends on Repository)

If your repository is private, you will need to give access to ArgoCD.  For this,
you will create a secret that contains a username and token to your repository.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argoproj-https-creds
  namespace: openshift-gitops
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: https://github.com/cmays20/OpenShift-GitOps-HomeLab.git
  type: git
  password: my-password
  username: my-username
```

## Order of Operations for Installing Operators

1. ArgoCD Permissions
2. Local Storage
3. ODF
4. Cert Manager
5. Internal Registry Setup
6. OpenShift Pipelines
7. AAP


## TODO

1. Tekton Results
2. Service Mesh 3
3. MTV
4. ACS
5. Compliance Operator
7. Web Terminal
8. Trusted Supply Chain
9. Logging